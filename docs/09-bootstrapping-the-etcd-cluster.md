# 9: Bootstrapping the etcd Cluster

In this lab, we will be configuring the `etcd` cluster for our Kubernetes masters to store their state in.

## Overview

The Kubernetes control plane is just a collection of services that relate to the cluster's operations and lifecycle. A cluster's primary component, the Kubernetes API server, requires a place to store information about the cluster state as it is modified by other services. Currently, `etcd` is the service responsible for persisting this state information.

[Etcd](https://github.com/coreos/etcd) is, in it's own words, a "distributed reliable key-value store for the most critical data of a distributed system". It is a clustered key-value database that is meant for storing system information. Modifications are done via a gRPC endpoint where applications such as the Kubernetes API server connect to. Much like Consul, `etcd` uses the same raft consensus algorithm to maintain this cluster state. A modified value is not consistent until all nodes in the cluster reach "consensus" on this changed value. This is also optionally true of reads from the cluster as well.

We will be installing the `etcd` server on each of our Kubernetes masters.

## NTP

A common problem when working with PKI is the time being incorrect on your machines. In the case of our vagrant machines, every time we suspend them, we effectively "stop the clock". This can ultimately lead to services such as `etcd` complaining that your certificates are not yet valid.

To fix this, we will install [NTP](http://www.ntp.org/) on each of our nodes so they can sync their clocks to the right time via the Internet:

Masters:

```bash
for i in $(seq 1 ${MASTER_NODE_COUNT}); do
vagrant ssh master${i} -- <<EOF
sudo apt-get update
sudo apt-get install ntp
EOF
done
```

Workers:

```bash
for i in $(seq 1 ${WORKER_NODE_COUNT}); do
vagrant ssh worker${i} -- <<EOF
sudo apt-get update
sudo apt-get install ntp
EOF
done
```

### Verification

Running the `date -R` command on any node should return the current time in UTC:

```bash
vagrant@master1:~$ date -R
Mon, 28 May 2018 17:51:43 +0000
```

## Etcd Static Pod

Much like Consul, we will be using a Static Pod to run the `etcd` cluster. We will start with creating a list of internal hostnames like we did in section 4, only this time it is formatted for `etcd`:

```bash
for i in $(seq 1 ${MASTER_NODE_COUNT}); do
  if [[ ${i} == 1 ]]; then
    export MASTER_LIST="master1=https://master1.node.local.consul:2380"
  else
    export MASTER_LIST="${MASTER_LIST},master${i}=https://master${i}.node.local.consul:2380"
  fi
done
```

This gives us a list similar to `master1=https://master1.node.local.consul:2380,master2=https://master2.node.local.consul:2380,master3=https://master3.node.local.consul:2380`. Now that consul is configured and running, we can use these DNS names to resolve to our master nodes.

**Note:** You may be wondering why we are not levering a Consul SRV record to use DNS discovery with `etcd`. Truth be told, I just couldn't get it working. No matter the setup, it would always only grab the first two nodes. If you have a configuration that works, please let me know!

We will create a different manifest for each node:

```bash
for i in $(seq 1 ${MASTER_NODE_COUNT}); do
export MASTER_IP=$(vagrant ssh master${i} -- ip -4 addr show dev eth1 | grep inet | tr -s " " | cut -d " " -f 3 | head -n 1 | cut -d "/" -f 1)
cat > master${i}-etcd.manifest <<EOF
apiVersion: v1
kind: Pod
metadata:
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ""
  labels:
    k8s-app: etcd-server
  name: etcd-server
  namespace: kube-system
spec:
  containers:
  - name: etcd
    image: quay.io/coreos/etcd:v3.3.5
    command:
    - /bin/sh
    - -c
    - /usr/local/bin/etcd 2>&1 | /usr/bin/tee -a /var/log/etcd.log
    env:
    - name: ETCD_NAME
      value: master${i}
    - name: ETCD_CERT_FILE
      value: /etc/etcd/kubernetes.pem
    - name: ETCD_KEY_FILE
      value: /etc/etcd/kubernetes-key.pem
    - name: ETCD_PEER_CERT_FILE
      value: /etc/etcd/kubernetes.pem
    - name: ETCD_PEER_KEY_FILE
      value: /etc/etcd/kubernetes-key.pem
    - name: ETCD_TRUSTED_CA_FILE
      value: /etc/etcd/ca.pem
    - name: ETCD_PEER_TRUSTED_CA_FILE
      value: /etc/etcd/ca.pem
    - name: ETCD_PEER_CLIENT_CERT_AUTH
      value: "true"
    - name: ETCD_CLIENT_CERT_AUTH
      value: "true"
    - name: ETCD_INITIAL_ADVERTISE_PEER_URLS
      value: "https://master${i}.node.local.consul:2380"
    - name: ETCD_ADVERTISE_CLIENT_URLS
      value: "https://master${i}.node.local.consul:2379"
    - name: ETCD_LISTEN_PEER_URLS
      value: "https://${MASTER_IP}:2380"
    - name: ETCD_LISTEN_CLIENT_URLS
      value: "https://${MASTER_IP}:2379,http://127.0.0.1:2379"
    - name: ETCD_INITIAL_CLUSTER_TOKEN
      value: etcd-cluster-0
    - name: ETCD_INITIAL_CLUSTER
      value: "${MASTER_LIST}"
    - name: ETCD_INITIAL_CLUSTER_STATE
      value: new
    - name: ETCD_DATA_DIR
      value: /var/lib/etcd
    livenessProbe:
      initialDelaySeconds: 15
      tcpSocket:
        host: 127.0.0.1
        port: 2379
      timeoutSeconds: 15
    ports:
    - containerPort: 2380
      hostPort: 2380
      name: serverport
    - containerPort: 2379
      hostPort: 2379
      name: clientport
    resources:
      requests:
        cpu: 200m
    volumeMounts:
    - mountPath: /etc/etcd
      name: etc-etcd
    - mountPath: /var/lib/etcd
      name: var-lib-etcd
    - mountPath: /var/log
      name: var-log
    - mountPath: /etc/hosts
      name: etc-hosts
      readOnly: true
  hostNetwork: true
  tolerations:
  - key: CriticalAddonsOnly
    operator: Exists
  volumes:
  - hostPath:
      path: /etc/etcd
    name: etc-etcd
  - hostPath:
      path: /var/lib/etcd
    name: var-lib-etcd
  - hostPath:
      path: /var/log
    name: var-log
  - hostPath:
      path: /etc/hosts
    name: etc-hosts
status: {}
EOF
done
```

The configuration values provided are very similar to those in our Consul configuration. The only major differences are that `etcd` uses TLS for node-to-node communications, which we will secure with the same certificate used for the API server. In addition, we configure the bind address of the `etcd` service, and provide the join list to tell it where to look for other nodes. We also mount `/var/lib/etcd` to store cluster state as well as `/etc/etcd` for configuration information. Both `/etc/hosts` and `/var/log` are mounted for access to system resources.

With the configuration files setup, we create the required directories and copy the certificates as well as the manifests to each master node:

```bash
for i in $(seq 1 ${MASTER_NODE_COUNT}); do
vagrant scp kubernetes.pem master${i}:~/kubernetes.pem
vagrant scp kubernetes-key.pem master${i}:~/kubernetes-key.pem
vagrant scp ca.pem master${i}:~/ca.pem
vagrant scp master${i}-etcd.manifest master${i}:~/etcd.manifest
vagrant ssh master${i} -- <<EOF
sudo mkdir -pv /etc/etcd
sudo chown -v root:root kubernetes.pem kubernetes-key.pem ca.pem etcd.manifest
sudo mv -v ./kubernetes.pem /etc/etcd/kubernetes.pem
sudo mv -v ./kubernetes-key.pem /etc/etcd/kubernetes-key.pem
sudo mv -v ./ca.pem /etc/etcd/ca.pem
sudo mv -v ./etcd.manifest /etc/kubernetes/manifests/etcd.manifest
EOF
done
```

### Verification

To verify that the cluster is working, we can run `etcdctl` in a Docker container:

```bash
sudo docker run --rm -it --net=host -e ETCDCTL_API=3 quay.io/coreos/etcd:v3.3.5 /usr/local/bin/etcdctl member list --endpoints=http://127.0.0.1:2379
3007515d972c83cf, started, master1, https://master1.node.local.consul:2380, https://master1.node.local.consul:2379
6b2bdb7c08468561, started, master2, https://master2.node.local.consul:2380, https://master2.node.local.consul:2379
9493170f835d6beb, started, master3, https://master3.node.local.consul:2380, https://master3.node.local.consul:2379
```

This should return a list of all master nodes.

Next: [Bootstrapping the etcd Cluster](09-bootstrapping-the-etcd-cluster.md)