# 6: Installing the Docker Container Runtime Interface (CRI)

In this lab, we will be installing the Container Runtime Interface, which is what Kubernetes uses to run containers.

## Overview

Kubernetes is based entirely around the idea of orchestrating containerized workloads. Before we can start running these workloads, we must have some sort of tool to actually run these containers. After all, Kubernetes is only an orchestrator. To actually run these containers, we will need what is called a Container Runtime Interface (CRI). At it's most basic, a CRI is a system daemon which exposes a socket that the kubelet can communicate with in order to manage containers on the system. For this lab, we will be using the [Docker](https://www.docker.com/) CRI.

## Docker

To install Docker, we will first download the packaged release to our host machine:

```bash
curl -o docker.tgz https://download.docker.com/linux/static/stable/x86_64/docker-17.03.2-ce.tgz
```

We will then copy the CRI to the master nodes, extract it, and move it to the proper directory:

```bash
for i in $(seq 1 ${MASTER_NODE_COUNT}); do
vagrant scp docker.tgz master${i}:~/
vagrant ssh master${i} -- <<EOF
tar -xvf docker.tgz
sudo chown -v root:root ./docker/*
sudo mv -v ./docker/* /usr/local/bin/
EOF
done
```

And the worker nodes:

```bash
for i in $(seq 1 ${WORKER_NODE_COUNT}); do
vagrant scp docker.tgz worker${i}:~/
vagrant ssh worker${i} -- <<EOF
tar -xvf docker.tgz
sudo chown -v root:root ./docker/*
sudo mv -v ./docker/* /usr/local/bin/
EOF
done
```

### Verification

For each node, you should see the following output:

```
docker/
docker/docker-containerd-ctr
docker/docker-proxy
docker/docker
docker/docker-containerd
docker/dockerd
docker/docker-init
docker/docker-containerd-shim
docker/docker-runc
changed ownership of './docker/docker' from vagrant:vagrant to root:root
changed ownership of './docker/docker-containerd' from vagrant:vagrant to root:root
changed ownership of './docker/docker-containerd-ctr' from vagrant:vagrant to root:root
changed ownership of './docker/docker-containerd-shim' from vagrant:vagrant to root:root
changed ownership of './docker/dockerd' from vagrant:vagrant to root:root
changed ownership of './docker/docker-init' from vagrant:vagrant to root:root
changed ownership of './docker/docker-proxy' from vagrant:vagrant to root:root
changed ownership of './docker/docker-runc' from vagrant:vagrant to root:root
'./docker/docker' -> '/usr/local/bin/docker'
'./docker/docker-containerd' -> '/usr/local/bin/docker-containerd'
'./docker/docker-containerd-ctr' -> '/usr/local/bin/docker-containerd-ctr'
'./docker/docker-containerd-shim' -> '/usr/local/bin/docker-containerd-shim'
'./docker/dockerd' -> '/usr/local/bin/dockerd'
'./docker/docker-init' -> '/usr/local/bin/docker-init'
'./docker/docker-proxy' -> '/usr/local/bin/docker-proxy'
'./docker/docker-runc' -> '/usr/local/bin/docker-runc'
```

## Systemd

Now that we have Docker installed, it's time to daemonize it so that it will run on boot. To do this, we will use the Debian init system `systemd`. Services within `systemd` are referred to as "Units".

First, create the Docker Unit configuration:

```bash
cat > docker.service <<EOF
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.io

[Service]
ExecStart=/usr/local/bin/docker daemon \
  --iptables=false \
  --ip-masq=false \
  --host=unix:///var/run/docker.sock \
  --log-level=error \
  --storage-driver=overlay
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

This file specifies several configuration options for the Docker daemon. Most importantly, it tells Docker NOT to modify iptables or perform any sort of "masquerade routing". These functions are provided by the `kube-proxy` as well as our Container Networking Interface, which will be explained in later sections. Finally, it tells Docker to expose a socket at `/var/run/docker.sock` so that the Kubelet can communicate with it.

With the file created, we will copy the Unit file to each master node, reload systemd's service catalog, and start the Docker service:

```bash
for i in $(seq 1 ${MASTER_NODE_COUNT}); do
vagrant scp docker.service master${i}:~/
vagrant ssh master${i} -- <<EOF
sudo chown -v root:root docker.service
sudo mv -v docker.service /etc/systemd/system/docker.service
sudo systemctl daemon-reload
sudo systemctl enable docker
sudo systemctl start docker
EOF
done
```

And on the worker nodes:

```bash
for i in $(seq 1 ${WORKER_NODE_COUNT}); do
vagrant scp docker.service worker${i}:~/
vagrant ssh worker${i} -- <<EOF
sudo chown -v root:root docker.service
sudo mv -v docker.service /etc/systemd/system/docker.service
sudo systemctl daemon-reload
sudo systemctl enable docker
sudo systemctl start docker
EOF
done
```

### Verification

On any node, run the `ps -aux | grep docker` command, and it should return the `docker` daemon's process information:

```bash
> vagrant ssh master1
Linux master1 4.9.0-6-amd64 #1 SMP Debian 4.9.82-1+deb9u3 (2018-03-02) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun May 27 00:55:53 2018 from 10.0.2.2
vagrant@master1:~$ ps -aux | grep docker
root     12828  0.1  3.4 317248 35240 ?        Ssl  00:57   0:00 dockerd --iptables=false --ip-masq=false --host=unix:///var/run/docker.sock --log-level=error --storage-driver=overlay
root     12838  0.0  0.8 137560  8704 ?        Ssl  00:57   0:00 docker-containerd -l unix:///var/run/docker/libcontainerd/docker-containerd.sock --metrics-interval=0 --start-timeout 2m --state-dir /var/run/docker/libcontainerd/containerd
--shim docker-containerd-shim --runtime docker-runc
vagrant  12939  0.0  0.0  12784  1020 pts/0    S+   01:00   0:00 grep docker
```

Next: [Installing the Kubernetes Node Agent (Kubelet)](07-installing-the-kubernetes-node-agent.md)