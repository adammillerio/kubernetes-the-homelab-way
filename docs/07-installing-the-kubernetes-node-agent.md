# 7: Installing the Kubernetes Node Agent (Kubelet)

In this lab, we will be installing the Kubelet, which is the system agent of Kubernetes.

## Overview

In a Kubernetes cluster, the Kubelet is the piece of software that makes an ordinary machine into a Kubernetes Node. Essentially, it is a system agent which connects to the Kubernetes control plane and uses information gained from the API to change what is running on the system. It is difficult to explain the Kubelet other than to say it is the main building block in a Kubernetes cluster, and is responsible for performing system functions such as mounting volumes as well as coordinating with the CRI to start new workloads.

## socat

The Kubernetes API provides the ability to setup port-forwards to workloads running in the cluster. In order to setup these forwarding channels, it relies on the `socat` tool. We will install that on the master nodes:

```bash
for i in $(seq 1 ${MASTER_NODE_COUNT}); do
vagrant ssh master${i} -- <<EOF
sudo apt-get update
sudo apt-get install socat
EOF
done
```

And on the worker nodes:

```bash
for i in $(seq 1 ${WORKER_NODE_COUNT}); do
vagrant ssh worker${i} -- <<EOF
sudo apt-get update
sudo apt-get install socat
EOF
done
```

### Verification

The `socat` binary should now be present on each node:

```bash
vagrant@master1:~$ socat
2018/05/27 01:33:31 socat[13283] E exactly 2 addresses required (there are 0); use option "-h" for help
```

## CNI Plugins

Another fundamental component of the Kubernetes cluster is the Container Networking Interface (CNI). This is used to provide an overlay network within the cluster so that Pods on different Nodes can communicate with one another via in-cluster IP addresses. This will be explained in greater detail in later sections.

For now, we will download the CNI plugin set to our host:

```bash
curl -L -o cni-plugins.tgz "https://github.com/containernetworking/plugins/releases/download/v0.6.0/cni-plugins-amd64-v0.6.0.tgz"
```

Then, we will copy the tar archive to the master nodes, create the CNI plugin directory, and extract the contents to it:

```bash
for i in $(seq 1 ${MASTER_NODE_COUNT}); do
vagrant scp cni-plugins.tgz master${i}:~/
vagrant ssh master${i} -- <<EOF
sudo mkdir -pv /opt/cni/bin/
sudo tar -xvf ./cni-plugins.tgz -C /opt/cni/bin/
EOF
done
```

And on the worker nodes:

```bash
for i in $(seq 1 ${WORKER_NODE_COUNT}); do
vagrant scp cni-plugins.tgz worker${i}:~/
vagrant ssh worker${i} -- <<EOF
sudo mkdir -pv /opt/cni/bin/
sudo tar -xvf ./cni-plugins.tgz -C /opt/cni/bin/
EOF
done
```

### Verification

For each node you should see the following output:

```
mkdir: created directory '/opt/cni'
mkdir: created directory '/opt/cni/bin/'
./
./flannel
./ptp
./host-local
./portmap
./tuning
./vlan
./sample
./dhcp
./ipvlan
./macvlan
./loopback
./bridge
```

## Disable Swap

Kubernetes has a hard requirement that swap be disabled on the system. We will do this on all nodes:

Master:

```bash
for i in $(seq 1 ${MASTER_NODE_COUNT}); do
vagrant ssh master${i} -- <<EOF
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
sudo swapoff -a
EOF
done
```

Worker:

```bash
for i in $(seq 1 ${WORKER_NODE_COUNT}); do
vagrant ssh worker${i} -- <<EOF
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
sudo swapoff -a
EOF
done
```

### Verification

Any swap partitions in `/etc/fstab` will now be commented out:

```
#UUID=7eaba3a9-be36-4e23-b5d6-80a87a436a8b none swap sw 0 0
```

## Kubelet

To install the Kubelet, we will first download it to our host machine:

```bash
curl -o kubelet https://storage.googleapis.com/kubernetes-release/release/v1.10.3/bin/linux/amd64/kubelet
```

We will then copy the kubelet to the master nodes and move it to the proper directory:

```bash
for i in $(seq 1 ${MASTER_NODE_COUNT}); do
vagrant scp kubelet master${i}:~/
vagrant ssh master${i} -- <<EOF
sudo chown -v root:root ./kubelet
sudo chmod +x ./kubelet
sudo mv -v ./kubelet /usr/local/bin/
EOF
done
```

And the worker nodes:

```bash
for i in $(seq 1 ${WORKER_NODE_COUNT}); do
vagrant scp kubelet worker${i}:~/
vagrant ssh worker${i} -- <<EOF
sudo chown -v root:root ./kubelet
sudo chmod +x ./kubelet
sudo mv -v ./kubelet /usr/local/bin/
EOF
done
```

### Verification

For each node you should see the following output:

```
changed ownership of './kubelet' from vagrant:vagrant to root:root
'./kubelet' -> '/usr/local/bin/kubelet'
```

## Kubelet Configuration

The Kubelet uses a standard Kubernetes configuration file.

First we will create the master configuration:

```bash
cat > master-kubelet-kubeconfig.yml <<EOF
apiVersion: v1
kind: Config
preferences: {}
current-context: service-account-context
contexts:
- context:
    cluster: local
    user: kubelet
  name: service-account-context
clusters:
- cluster:
    server: http://localhost:8080
  name: local
users:
- name: kubelet
  user:
    as-user-extra: {}
    client-certificate: /var/lib/kubelet/client.pem
    client-key: /var/lib/kubelet/client-key.pem
EOF
```

This configuration file tells the Kubelet where to look for a Kubernetes API server, in the case of the masters this is localhost, as it will eventually have an API server present. In addition, it specifies that it will authenticate using the client certificate and key located at `/var/lib/kubelet/`.

We will then copy this configuration file, the master certificate and key, and the CA certificate to the machine. Then, we will move them to their respective locations:

```bash
for i in $(seq 1 ${MASTER_NODE_COUNT}); do
vagrant scp ca.pem master${i}:~/ca.pem
vagrant scp master${i}.pem master${i}:~/client.pem
vagrant scp master${i}-key.pem master${i}:~/client-key.pem
vagrant scp master-kubelet-kubeconfig.yml master${i}:~/kubeconfig
vagrant ssh master${i} -- <<EOF
sudo mkdir -pv /var/lib/kubernetes
sudo chown -v root:root ./ca.pem
sudo mv -v ./ca.pem /var/lib/kubernetes/ca.pem
sudo mkdir -pv /var/lib/kubelet
sudo chown -v root:root ./client.pem ./client-key.pem
sudo mv -v ./client.pem /var/lib/kubelet/client.pem
sudo mv -v ./client-key.pem /var/lib/kubelet/client-key.pem
sudo chown -v root:root ./kubeconfig
sudo mv -v ./kubeconfig /var/lib/kubelet/kubeconfig
EOF
done
```

Now, we will create a configuration for the worker nodes:

```bash
cat > worker-kubelet-kubeconfig.yml <<EOF
apiVersion: v1
kind: Config
preferences: {}
current-context: service-account-context
contexts:
- context:
    cluster: local
    user: kubelet
  name: service-account-context
clusters:
- cluster:
    certificate-authority: /var/lib/kubernetes/ca.pem
    server: https://kubernetes.service.consul:6443
  name: local
users:
- name: kubelet
  user:
    as-user-extra: {}
    client-certificate: /var/lib/kubelet/client.pem
    client-key: /var/lib/kubelet/client-key.pem
EOF
```

The worker configuration is similar to that of the master, with one difference. The API service will not be available on the worker nodes, so it will need to reach out to the masters to join the cluster. To do this, we will be using Consul for service discovery, but this will be covered in the next section. Just know that the DNS name configured here will eventually resolve to the cluster masters. In addition, it will connect to these hosts over an encrypted connection for security. Because of this, we have to specify the CA certificate to use to verify the remote master.

Now, we will perform the same steps on the workers:

```bash
for i in $(seq 1 ${WORKER_NODE_COUNT}); do
vagrant scp ca.pem worker${i}:~/ca.pem
vagrant scp worker${i}.pem worker${i}:~/client.pem
vagrant scp worker${i}-key.pem worker${i}:~/client-key.pem
vagrant scp worker-kubelet-kubeconfig.yml worker${i}:~/kubeconfig
vagrant ssh worker${i} -- <<EOF
sudo mkdir -pv /var/lib/kubernetes
sudo chown -v root:root ./ca.pem
sudo mv -v ./ca.pem /var/lib/kubernetes/ca.pem
sudo mkdir -pv /var/lib/kubelet
sudo chown -v root:root ./client.pem ./client-key.pem
sudo mv -v ./client.pem /var/lib/kubelet/client.pem
sudo mv -v ./client-key.pem /var/lib/kubelet/client-key.pem
sudo chown -v root:root ./kubeconfig
sudo mv -v ./kubeconfig /var/lib/kubelet/kubeconfig
EOF
done
```

### Verification

For each node you should see the following output:

```
mkdir: created directory '/var/lib/kubernetes'
changed ownership of './ca.pem' from vagrant:vagrant to root:root
'./ca.pem' -> '/var/lib/kubernetes/ca.pem'
mkdir: created directory '/var/lib/kubelet'
changed ownership of './client.pem' from vagrant:vagrant to root:root
changed ownership of './client-key.pem' from vagrant:vagrant to root:root
'./client.pem' -> '/var/lib/kubelet/client.pem'
'./client-key.pem' -> '/var/lib/kubelet/client-key.pem'
changed ownership of './kubeconfig' from vagrant:vagrant to root:root
'./kubeconfig' -> '/var/lib/kubelet/kubeconfig'
```

## Systemd

Much like we did for Docker, we will create a `systemd` Unit for the kubelet in order to daemonize it:

Master:

```bash
for i in $(seq 1 ${MASTER_NODE_COUNT}); do
cat > master${i}-kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --allow-privileged=true \\
  --anonymous-auth=false \\
  --authorization-mode=Webhook \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --cloud-provider= \\
  --cluster-dns=100.64.0.10 \\
  --cluster-domain=cluster.local \\
  --container-runtime=docker \\
  --docker=unix:///var/run/docker.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --cni-bin-dir=/opt/cni/bin/ \\
  --cni-conf-dir=/etc/cni/net.d/ \\
  --non-masquerade-cidr=100.64.0.0/10 \\
  --register-node=true \\
  --runtime-request-timeout=15m \\
  --tls-cert-file=/var/lib/kubelet/client.pem \\
  --tls-private-key-file=/var/lib/kubelet/client-key.pem \\
  --hostname-override=master${i} \\
  --pod-manifest-path=/etc/kubernetes/manifests \\
  --eviction-hard=memory.available<100Mi,nodefs.available<10%,nodefs.inodesFree<5%,imagefs.available<10%,imagefs.inodesFree<5% \\
  --feature-gates=ExperimentalCriticalPodAnnotation=true \\
  --node-labels=kubernetes.io/role=master,node-role.kubernetes.io/master= \\
  --register-with-taints=node-role.kubernetes.io/master=:NoSchedule \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
done
```

TODO: Explain this

Worker:

```bash
for i in $(seq 1 ${WORKER_NODE_COUNT}); do
cat > worker${i}-kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --allow-privileged=true \\
  --anonymous-auth=false \\
  --authorization-mode=Webhook \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --cloud-provider= \\
  --cluster-dns=100.64.0.10 \\
  --cluster-domain=cluster.local \\
  --container-runtime=docker \\
  --docker=unix:///var/run/docker.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --cni-bin-dir=/opt/cni/bin/ \\
  --cni-conf-dir=/etc/cni/net.d/ \\
  --non-masquerade-cidr=100.64.0.0/10 \\
  --register-node=true \\
  --register-schedulable=true \\
  --runtime-request-timeout=15m \\
  --tls-cert-file=/var/lib/kubelet/client.pem \\
  --tls-private-key-file=/var/lib/kubelet/client-key.pem \\
  --hostname-override=worker${i} \\
  --pod-manifest-path=/etc/kubernetes/manifests \\
  --eviction-hard=memory.available<100Mi,nodefs.available<10%,nodefs.inodesFree<5%,imagefs.available<10%,imagefs.inodesFree<5% \\
  --feature-gates=ExperimentalCriticalPodAnnotation=true \\
  --node-labels=kubernetes.io/role=node,node-role.kubernetes.io/node= \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
done
```

With the files created, we will copy the Unit files to each master node, reload systemd's service catalog, and start the Kubelet service:

```bash
for i in $(seq 1 ${MASTER_NODE_COUNT}); do
vagrant scp master${i}-kubelet.service master${i}:~/kubelet.service
vagrant ssh master${i} -- <<EOF
sudo mkdir -pv /etc/kubernetes/manifests
sudo chown -v root:root kubelet.service
sudo mv -v kubelet.service /etc/systemd/system/kubelet.service
sudo systemctl daemon-reload
sudo systemctl enable kubelet
sudo systemctl start kubelet
EOF
done
```

And on the worker nodes:

```bash
for i in $(seq 1 ${WORKER_NODE_COUNT}); do
vagrant scp worker${i}-kubelet.service worker${i}:~/kubelet.service
vagrant ssh worker${i} -- <<EOF
sudo mkdir -pv /etc/kubernetes/manifests
sudo chown -v root:root kubelet.service
sudo mv -v kubelet.service /etc/systemd/system/kubelet.service
sudo systemctl daemon-reload
sudo systemctl enable kubelet
sudo systemctl start kubelet
EOF
done
```

### Verification

On any node, run the `ps -aux | grep kubelet` command, and it should return the `kubelet` daemon's process information:

```
vagrant@master1:~$ ps -aux | grep kubelet
root     14099  0.7  7.0 482672 71924 ?        Ssl  02:18   0:05 /usr/local/bin/kubelet --allow-privileged=true --anonymous-auth=false --authorization-mode=Webhook --client-ca-file=/var/lib/kubernetes/ca.pem --cloud-provider= --cluster-dns=100.64.0.10 --cluster-domain=cluster.local --container-runtime=docker --docker=unix:///var/run/docker.sock --image-pull-progress-deadline=2m --kubeconfig=/var/lib/kubelet/kubeconfig --network-plugin=cni --cni-bin-dir=/opt/cni/bin/ --cni-conf-dir=/etc/cni/net.d/ --non-masquerade-cidr=100.64.0.0/10 --register-node=true --runtime-request-timeout=15m --tls-cert-file=/var/lib/kubelet/client.pem --tls-private-key-file=/var/lib/kubelet/client-key.pem --hostname-override=master1 --pod-manifest-path=/etc/kubernetes/manifests --eviction-hard=memory.available<100Mi,nodefs.available<10%,nodefs.inodesFree<5%,imagefs.available<10%,imagefs.inodesFree<5 --feature-gates=ExperimentalCriticalPodAnnotation=true --node-labels=kubernetes.io/role=master,node-role.kubernetes.io/master= --register-with-taints=node-role.kubernetes.io/master=:NoSchedule --v=2
```

**Note:** The kubelet is likely printing a ton of errors to the system journal at this point. This is because it has no API service to talk to right now. This will be resolved in later sections.

Next: [Configuring Node Auto-Discovery](08-configuring-node-auto-discovery.md)