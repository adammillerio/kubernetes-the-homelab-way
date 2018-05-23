# Installing the Client Tools

In this lab, we will be installing the host tools that will be necessary to provision the K8S cluster compute resources in Section 2.

## VirtualBox

[Oracle VM VirtualBox](https://www.virtualbox.org/) is a common virtualization platform that is available on all major operating systems. There are many different VM tools to choose from, but this tutorial will use VirtualBox since it is both free and cross-platform. In addition, the Vagrant tooling described below will target VirtualBox as well.

To install VirtualBox, navigate to the [VirtualBox website](https://www.virtualbox.org/) and install the version for your platform.

### Verification

After installation, the VirtualBox tool should now be available.

## vagrant

This tutorial will make use of [Vagrant](https://www.vagrantup.com/) which is a tool that automates the creation of Virtual Machines. While it is of course optional to make use of the Vagrant tools included in this tutorial, it will assume that you are using them.

To install Vagrant, simply navigate to the [Vagrant website](https://www.vagrantup.com/) and install the version for your platform.

### Verification

The `vagrant version` command should now output it's version if it is in your path:

```bash
> vagrant version
Installed Version: 2.1.1
Latest Version: 2.1.1
```

Next: [Provisioning Compute Resources](03-provisioning-compute-resources.md)