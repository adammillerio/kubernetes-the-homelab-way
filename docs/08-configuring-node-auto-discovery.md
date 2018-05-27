# 8: Configuring Node Auto-Discovery

In this lab, we will be configuring a mechanism to allow nodes to automatically discover one another on the network.

## Overview

Our Kubernetes cluster is expected to always have three master nodes. Because of this, we are able to reasonably assume that they will always be available at the same IP address. However, our worker nodes are expected to live and die as they are needed. With this in mind, how do we allow our workers to discover the Kubernetes cluster?

To accomplish this, we will be using [Consul](https://www.consul.io/) which is a clustered service discovery application. Consul is very similar to the `etcd` state store we will be configuring for Kubernetes. It consists of one or more servers which keep track of all nodes within a Consul cluster, as well as what services they expose.

We will configure the master nodes to all serve as Consul "servers". These nodes will use the [Raft Consensus Algorithm](https://raft.github.io/) to collectively maintain cluster state. In addition, we will configure our worker nodes to register themselves to this cluster. Finally, we will create a Kubernetes API service within our cluster that we expose to our workers so they always know where to look to find the API.

In section 7, we configured the Kubelet to attempt to reach the cluster at `https://kubernetes.service.consul:6443` by the end of this section, we will have a dynamically maintained set of endpoints that the worker nodes will be able to find at this address.

## Consul Static Pod

Based on the previous sections, it would be reasonable to assume that we will be running Consul using another `systemd` Unit file. However, we will instead be leveraging a feature within Kubernetes called [Static Pods](https://kubernetes.io/docs/tasks/administer-cluster/static-pod/). Essentially, a Static Pod is a Kubernetes Pod which is managed locally by the kubelet instead of by the cluster itself.

When we created our `kubelet` configuration, we specified the `--pod-manifest-path=/etc/kubernetes/manifests` option. This tells the kubelet to look in this folder for any Pod manifest files on startup. If it finds any, it will then use the manifest to create a Pod on the node itself and manage it as if it were any other Pod in the cluster. This brings with it the following benefits:

* Allows for running critical system services as Docker containers
* The kubelet is able to manage these services like any other Kubernetes workload
* The service will be visible at a "cluster-level" allowing for the ability to port forward and view logs as if they were actual Pods in the cluster

It is important to note that while these Pods may appear to be in the cluster, they cannot be controlled at a cluster-level. For example, if you were to run the `kubectl delete pod` command on one of these Pods, nothing would happen, as only the kubelet has control of it.

We will be using Static Pods to configure all of our core Kubernetes services. In the case of Consul, we will be using it to run the Consul Docker container.

First, we create the "join list" which is essentially a list of known IP addresses of master nodes that the Consul agent should connect to when trying to form or join a cluster:

```bash
for i in $(seq 1 ${MASTER_NODE_COUNT}); do
  export MASTER_IP=$(vagrant ssh master${i} -- ip -4 addr show dev eth1 | grep inet | tr -s " " | cut -d " " -f 3 | head -n 1 | cut -d "/" -f 1)

  if [[ ${i} == 1 ]]; then
    export JOIN_LIST="-retry-join=${MASTER_IP}"
  else
    export JOIN_LIST="${JOIN_LIST} -retry-join=${MASTER_IP}"
  fi
done
```

This aggregates the IP addresses of our master nodes' host only adapters and creates a string similar to `-retry-join=10.0.0.11 -retry-join=10.0.0.12 -retry-join=10.0.0.13` which essentially tells the Consul agent to look for other Consul agents at those IP addresses when it boots. This allows nodes in our cluster to be able to connect back to any server in the event of failure.

Then, create the Pod manifest for the Consul masters:

```bash
cat > consul-master.manifest <<EOF
apiVersion: v1
kind: Pod
metadata:
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ""
  labels:
    k8s-app: consul-master
  name: consul-master
  namespace: kube-system
spec:
  containers:
  - name: consul
    image: consul:1.1.0
    command:
    - /bin/sh
    - -c
    - /bin/consul
      agent
      -server
      -datacenter="local"
      -dns-port=53
      -data-dir="/consul/data"
      -config-dir="/consul/config"
      -bootstrap-expect=${MASTER_NODE_COUNT}
      ${JOIN_LIST}
      -bind='{{ GetInterfaceIP "eth1" }}' 2>&1 | /usr/bin/tee -a /var/log/consul.log
    livenessProbe:
      httpGet:
        host: 127.0.0.1
        path: /
        port: 8500
      initialDelaySeconds: 15
      timeoutSeconds: 15
    resources:
      requests:
        cpu: 100m
    volumeMounts:
    - mountPath: /consul/data
      name: consul-data
    - mountPath: /consul/config
      name: consul-config
    - mountPath: /var/log
      name: var-log
  hostNetwork: true
  tolerations:
  - key: CriticalAddonsOnly
    operator: Exists
  volumes:
  - hostPath:
      path: /opt/consul/data
    name: consul-data
  - hostPath:
      path: /opt/consul/config
    name: consul-config
  - hostPath:
      path: /var/log
    name: var-log
status: {}
EOF
```

This is the first Kubernetes manifest that we have encountered in this tutorial. This is no different from any other Kubernetes manifest, other than that it is installed directly on the master nodes as a Static Pod. This creates a Pod in the kube-system namespace with the name consul. Within it, an instance of the consul agent is ran in server mode. The server is configured to not attempt to bootstrap the cluster until it sees that all masters have connected. In addition, it mounts the directories under `/opt/consul` into the container so that state can be persisted across reboots. The Pod is also configured to use host networking, so that it can listen directly on the node's bind address without relying on Docker networking. This allows the Consul DNS service to listen on the standard DNS port 53 on our machine, which we will leverage for discovering Kubernetes API servers.

You may notice the "tolerations" section of this configuration, this is a section that essentially tells the Pod to "tolerate" certain annotations on a Node. In this instance, the master node is set with the CriticalAddonsOnly annotation, to prevent Pods not specifically used for the control plane from being scheduled onto our master nodes. There is also a "liveness probe" setup so that the kubelet can check whether or not the Consul service is running, as well as a resource limit of 100 "millicpus" to prevent it from consuming too many resources.

Next, create the kubernetes service definition for Consul:

```bash
cat > consul-kubernetes-service.json <<EOF
{
  "service": {
    "name": "kubernetes",
    "tags": ["kubernetes"],
    "port": 6443
  }
}
EOF
```

This tells consul that it should expose a Service at `kubernetes.service.consul` which listens on port 6443 and register the node to it. This will be the service that our workers use in order to figure out where to go to find the Kubernetes API.

Now, we will create the data and configuration directories and copy the manifest and service to each master:

```bash
for i in $(seq 1 ${MASTER_NODE_COUNT}); do
vagrant scp consul-master.manifest master${i}:~/consul.manifest
vagrant scp consul-kubernetes-service.json master${i}:~/kubernetes.json
vagrant ssh master${i} -- <<EOF
sudo mkdir -pv /opt/consul/data /opt/consul/config
sudo chown -v root:root ./consul.manifest ./kubernetes.json
sudo mv -v ./kubernetes.json /opt/consul/config/kubernetes.json
sudo mv -v ./consul.manifest /etc/kubernetes/manifests/consul.manifest
EOF
done
```

Now, we will move onto the worker nodes. We create a separate Pod manifest for these nodes:

```bash
cat > consul-worker.manifest <<EOF
apiVersion: v1
kind: Pod
metadata:
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ""
  labels:
    k8s-app: consul-worker
  name: consul-worker
  namespace: kube-system
spec:
  containers:
  - name: consul
    image: consul:1.1.0
    command:
    - /bin/sh
    - -c
    - /bin/consul
      agent
      -datacenter="local"
      -dns-port=53
      -data-dir="/consul/data"
      -config-dir="/consul/config"
      ${JOIN_LIST}
      -bind='{{ GetInterfaceIP "eth1" }}' 2>&1 | /usr/bin/tee -a /var/log/consul.log
    livenessProbe:
      httpGet:
        host: 127.0.0.1
        path: /
        port: 8500
      initialDelaySeconds: 15
      timeoutSeconds: 15
    resources:
      requests:
        cpu: 100m
    volumeMounts:
    - mountPath: /consul/data
      name: consul-data
    - mountPath: /consul/config
      name: consul-config
    - mountPath: /var/log
      name: var-log
  hostNetwork: true
  tolerations:
  - key: CriticalAddonsOnly
    operator: Exists
  volumes:
  - hostPath:
      path: /opt/consul/data
    name: consul-data
  - hostPath:
      path: /opt/consul/config
    name: consul-config
  - hostPath:
      path: /var/log
    name: var-log
status: {}
EOF
```

This manifest is very similar to the one for the masters. The only major difference is that the worker nodes do not run Consul in server mode. This makes it so that these nodes are able to join the cluster, but are not involved in the management of the state of the cluster. This makes sense, as the masters are the only nodes which are expected to be consistently available, so we do not want to store critical cluster information on a worker node or make it into a cluster leader.

Now, we will create the data and configuration directories and copy the manifest to each worker:

```bash
for i in $(seq 1 ${WORKER_NODE_COUNT}); do
vagrant scp consul-worker.manifest worker${i}:~/consul.manifest
vagrant ssh worker${i} -- <<EOF
sudo mkdir -pv /opt/consul/data /opt/consul/config
sudo chown -v root:root ./consul.manifest
sudo mv -v ./consul.manifest /etc/kubernetes/manifests/consul.manifest
EOF
done
```

Note that we do not copy the service definition to the workers, as they will not have the Kubernetes API service present on them.

### Verification

To verify that the Consul cluster is active, we can run the Consul administration command through a docker container on any master node:

```bash
sudo docker run --rm --net=host consul:1.1.0 members
Node     Address         Status  Type    Build  Protocol  DC     Segment
master1  10.0.0.11:8301  alive   server  1.1.0  2         local  <all>
master2  10.0.0.12:8301  alive   server  1.1.0  2         local  <all>
master3  10.0.0.13:8301  alive   server  1.1.0  2         local  <all>
worker1  10.0.0.21:8301  alive   client  1.1.0  2         local  <default>
worker2  10.0.0.22:8301  alive   client  1.1.0  2         local  <default>
```

In addition, we can verify that the Kubernetes service is active by installing and running the `dig` command:

```bash
sudo apt-get update
sudo apt-get install dnsutils
dig @127.0.0.1 kubernetes.service.consul
; <<>> DiG 9.10.3-P4-Debian <<>> @127.0.0.1 kubernetes.service.consul
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 35788
;; flags: qr aa rd; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;kubernetes.service.consul.     IN      A

;; ANSWER SECTION:
kubernetes.service.consul. 0    IN      A       10.0.0.11
kubernetes.service.consul. 0    IN      A       10.0.0.12
kubernetes.service.consul. 0    IN      A       10.0.0.13

;; Query time: 0 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Sun May 27 18:05:16 GMT 2018
;; MSG SIZE  rcvd: 102
```

The Consul DNS resolver is now providing our three master IPs as endpoints for the Kubernetes service.

## resolv.conf

Now that we have the Consul DNS service listening on port 53, we need to modify our Linux system to query this server by default. This is done via the `/etc/resolve.conf` file, which contains a list of nameservers to use in sequential order.

Unfortunately, the method of doing this differs significantly between operating systems, as this file is typically managed by other services. Luckily, if you are following this tutorial using `vagrant`, this file is being modified as a result of an inline shell provisioner found in the Vagrantfile:

```
  config.vm.provision "shell",
    inline: "sed -i '1s/^/nameserver 127.0.0.1 \\n/' /etc/resolv.conf",
    run: "always",
    privileged: true
```

This runs a sed command which adds the line `nameserver 127.0.0.1` to the top of the configuration file. This ensures that DNS resolution will attempt to use the Consul DNS server before reaching out to the Vagrant-managed DNS proxy to your host machine.

There are many ways to go about this, and if you are not using `vagrant` some additional research may be required.

### Verification

Now that the Consul DNS server is the primary resolver, we can run a similar dig command, without explicitly specifying what server to use. This should return our master endpoints:

```bash
vagrant@worker2:~$ dig kubernetes.service.consul
; <<>> DiG 9.10.3-P4-Debian <<>> kubernetes.service.consul
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 58586
;; flags: qr aa rd; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;kubernetes.service.consul.     IN      A

;; ANSWER SECTION:
kubernetes.service.consul. 0    IN      A       10.0.0.12
kubernetes.service.consul. 0    IN      A       10.0.0.11
kubernetes.service.consul. 0    IN      A       10.0.0.13

;; Query time: 1 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Sun May 27 20:42:19 GMT 2018
;; MSG SIZE  rcvd: 102
```

Next: [Bootstrapping the etcd Cluster](09-bootstrapping-the-etcd-cluster.md)