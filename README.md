# Learn Kubernetes the Homelab Way: Introduction

This is a series of tutorials for building a Kubernetes (K8S) cluster from scratch in a "bare-metal" homelab environment. It is heavily inspired by Kelsey Hightower's excellent ["Learn Kubernetes the Hard Way"](https://github.com/kelseyhightower/kubernetes-the-hard-way) and is designed in a very similar fashion.

## Ansible Roles

Before writing this tutorial, I created two separate [Ansible](https://www.ansible.com/) roles for configuring Kubernetes masters and workers respectively. These are heavily documented, and will essentially automatically configure nodes in a manner similar to this tutorial, they can be found at the following links:

* [ansible-role-k8s-master](https://github.com/adammillerio/ansible-role-k8s-master)
* [ansible-role-k8s-worker](https://github.com/adammillerio/ansible-role-k8s-worker)

## Table of Contents

* [Introduction](README.md)
* [Installing the Host Tools](docs/01-installing-the-host-tools.md)
* [Configuring Environment Variables](docs/02-configuring-environment-variables.md)
* [Provisioning Compute Resources](docs/03-provisioning-compute-resources.md)
* [Provisioning the CA and Generating TLS Certificates](docs/04-provisioning-the-ca-and-generating-tls-certificates.md)
* [Generating the Data Encryption Config and Key](docs/05-generating-the-data-encryption-config-and-key.md)
* Installing the Docker Container Runtime Interface (CRI)
* Provisioning the Kubernetes Masters
	* Installing the Kubernetes Node Agent (Kubelet)
	* Bootstrapping the etcd Cluster
	* Bootstrapping the Kubernetes Control Plane
	* Installing the flannel Container Network Interface (CNI)
	* Installing the Kubernetes DNS Addon (kube-dns)
* Provisioning the Kubernetes Workers
* Configuring Kubectl for Remote Access
* Smoke Test

## Motivation

As a member of a DevOps team, I've grown to love Kubernetes for its ability to allow for declarative infrastructure through Docker containers. However, there are a few things that I have noticed while working and interacting with K8S:

Because of it's microservice approach to infrastructure, the "stack" that K8S is comprised of is much less of a monolith than most traditional systems architecture, encompassing many different components which work together to form a cohesive whole. While many developers and engineers work with and interact with K8S on a daily basis, they do not understand how to actually build a cluster from scratch.

Having recently noticed that I was guilty of this, I set out to build a Kubernetes cluster on my own without leaning on any sort of managed service. To accomplish this, I began to work through Kelsey Hightower's excellent [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) tutorial. However, I didn't want to lean on any sort of cloud provider to provide this infrastructure. Instead, I wanted to build a cluster entirely on bare metal in my homelab. Because this approach caused me to diverge significantly from Kelsey's guide, I figured that I would write out my own tutorial for anyone else who is interested in doing the same thing.

## Prerequisites & Assumptions

* While I have tried to be as verbose as possible in explaing the various components of K8S, I will not be going into detail on Docker and containers in general, please refer to other tutorials to learn about them.
* I wrote this using Debian 9 "Stretch" as a base. This is not a hard requirement, but certain sections of the tutorial will make use of `apt` for package management as well as `systemd` for running K8S services.
* This tutorial targets x86_64 architectures. While K8S does run on ARM based systems, I have not attempted this. If you do, I encourage you to modify these automations for your own purposes.
* This tutorial uses `vagrant` to provision compute resources for this cluster, however, any systems that meet the requirements above should work. This tutorial will create the following resources:
	* 3 K8S master nodes
	* 2 K8S worker nodes
* K8S assumes that all nodes reside on the same "flat" network, where each one is capable of reaching one another. In addition, each node must have a fully routable dedicated IP address within this network. The `vagrant` setup ensures this, however, if you use your own resources, ensure that your network meets these requirements, or you will quickly run into issues with cluster communication.
