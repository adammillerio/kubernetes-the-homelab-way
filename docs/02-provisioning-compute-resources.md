# 2: Provisioning Compute Resources

In this lab, we will be provisioning the local compute resources that will be used for our K8S cluster.

## Overview

Kubernetes is a distributed system. This means that it is comprised of a set of "nodes" which all form a cohesive cluster. Fundamentally, what makes a machine a Kubernetes node is the presence of the `kubelet`. This is the agent which controls the actual system functions of a cluster, such as communicating with the Container Runtime (docker) to create and destroy containers.

In our cluster design, there will be no difference in the way in which the nodes themselves are provisioned and introduced into the cluster. That being said, there will be two different roles that nodes in our cluster assume. The first role is the worker node, which is responsible for actually running our containerized workloads. The second is the master node, which is a Kubernetes node that is dedicated to running the "control plane", which is a set of services crucial to cluster operations.

## Architecture

This tutorial is going to deploy the absolute minimum amount of nodes required for a High Availability (HA) configuration. That is three master nodes and two worker nodes. It is not possible to only have two master nodes, as one of the core components of the cluster, `etcd`, is quorum-based. This will be explained in greater detail later, but know that it is impossible to have a majority agreement in a quorum-based system if there are only two nodes, as you can never go beyond 50% confidence. These machines can be of any type, however, they must all share a private network and be able to communicate with one another. This is how the `kubelet` communicates to the `kube-apiserver` in the control plane to signify that it is ready to take workloads.

## Vagrant

This tutorial will be using VirtualBox as well as Vagrant to bootstrap a set of compute resources to be used as Kubernetes nodes. First, clone this repository to your machine. Then, edit the `vagrant_config.yml` file in the root of the repository. It should look something like this:

```yaml
---
config:
  box: "debian/contrib-stretch64"
  network: 10.0.0.0
  master:
    count: 3
    cpu: 2
    memory: 1024
  worker:
    count: 2
    cpu: 2
    memory: 1024
  client:
    enabled: true
    cpu: 1
    memory: 512
```

This file provides a set of configuration values that will be used in the corresponding [Vagrantfile](../Vagrantfile) to automatically bootstrap the compute resources. A brief explanation of each value is below:

| Name  | Default  | Description  |
|---|---|---|
| box | "debian/contrib-stretch64" | The Box, or Base Image, to be used; this tutorial targets Debian Stretch |
| network | 10.0.0.0 | The private network to place all nodes on; this must not overlap with your existing lab network |
| master.count | 3 | How many master nodes will be provisioned |
| master.cpu | 2 | Numer of vCPUs that will be assigned to each master node |
| master.memory | 1024 | Memory (in megabytes) that will be assigned to each master node |
| worker.count | 3 | How many worker nodes will be provisioned |
| worker.cpu | 2 | Numer of vCPUs that will be assigned to each worker node |
| worker.memory | 1024 | Memory (in megabytes) that will be assigned to each worker node |
| client.enabled | true | Whether or not the "client" machine is to be provisioned |
| client.cpu | 1 | Number of vCPUs that will be assigned to the client node |
| client.memory | 512 | Memory (in megabytes) that will be assigned to the client node |

The client machine is an optional component that will serve as a reference Linux machine to work through this tutorial on. The tutorial will assume you are using the client machine, but the same steps can be performed via the host machine easily.

I would strongly recommend that you modify these values so that they do not over-provision the resources available on your local machine. For example, in this reference configuration, the following resources will be used:

* (3 * 1024) + (2 * 1024) + (1 * 512) = 5656MB of RAM
* (3 * 2) + (2 * 2) + (1 * 1) = 11 vCPUs

After configuring the Vagrant setup to your liking, run the `vagrant up` command. This will perform the following steps for each node:

* Copy the base box image
* Boot the virtual machine
* Configure a second host-only network adapter on the private network as configured
* Set the hostname
* Install the `avahi` daemon to enable mDNS discovery of other nodes in the cluster

The purpose of mDNS is to allow for the use of a `.local` subdomain for reaching other nodes in the cluster. For example, to reach master2, instead of connecting to `10.0.0.11` we can connect to `master2.local` instead.

### Verification

After the cluster is up and running, the `vagrant status` command will show all of the nodes in the cluster. They should all be in the "running" status.

```bash
> vagrant status
Current machine states:

master1                   running (virtualbox)
master2                   running (virtualbox)
master3                   running (virtualbox)
worker1                   running (virtualbox)
worker2                   running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
```

To verify the private network is functioning correctly, ssh into the first master node and attempt to ping the first worker node at `worker1.local`:

```bash
> vagrant ssh master1
Linux master1 4.9.0-6-amd64 #1 SMP Debian 4.9.82-1+deb9u3 (2018-03-02) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Thu May 24 03:26:46 2018 from 10.0.2.2
vagrant@master1:~$ ping -c 3 worker1.local
PING worker1.local (10.0.0.21) 56(84) bytes of data.
64 bytes from 10.0.0.21 (10.0.0.21): icmp_seq=1 ttl=64 time=0.169 ms
64 bytes from 10.0.0.21 (10.0.0.21): icmp_seq=2 ttl=64 time=0.228 ms
64 bytes from 10.0.0.21 (10.0.0.21): icmp_seq=3 ttl=64 time=0.252 ms

--- worker1.local ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2019ms
rtt min/avg/max/mdev = 0.169/0.216/0.252/0.036 ms
```

If you are able to ping the worker node, move on to the next section.

Next: [Installing the Client Tools](03-installing-the-client-tools.md)