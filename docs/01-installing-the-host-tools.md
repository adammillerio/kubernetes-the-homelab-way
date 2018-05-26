# Installing the Host Tools

In this lab, we will be installing the host tools that will be necessary to provision the K8S cluster compute resources in Section 2. In addition, we will be installing the tools necessary to administrate the Kubernetes cluster.

## VirtualBox

[Oracle VM VirtualBox](https://www.virtualbox.org/) is a common virtualization platform that is available on all major operating systems. There are many different VM tools to choose from, but this tutorial will use VirtualBox since it is both free and cross-platform. In addition, the Vagrant tooling described below will target VirtualBox as well.

To install VirtualBox, navigate to the [VirtualBox website](https://www.virtualbox.org/) and install the version for your platform.

### Verification

After installation, the VirtualBox tool should now be available.

## vagrant

This tutorial will make use of [Vagrant](https://www.vagrantup.com/) which is a tool that automates the creation of Virtual Machines. While it is of course optional to make use of the Vagrant tools included in this tutorial, it will assume that you are using them.

To install Vagrant, simply navigate to the [Vagrant website](https://www.vagrantup.com/) and install the version for your platform.

After vagrant has been installed, the `vagrant scp` plugin will need to be installed as well:

```bash
vagrant plugin install vagrant-scp
```

### Verification

The `vagrant version` command should now output it's version if it is in your path:

```bash
> vagrant version
Installed Version: 2.1.1
Latest Version: 2.1.1
```

In addition, the `vagrant scp` command should function:

```bash
> vagrant scp
Usage: vagrant scp <local_path> [vm_name]:<remote_path>
       vagrant scp [vm_name]:<remote_path> <local_path>

Options:

    -h, --help                       Print this help
```

## cfssl & cfssljson

The `cfssl` and `cfssljson` tools will be used in order to generate the [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) for our K8S cluster. This will be described in greater detail in the section 3.

These tools are available from the [cfssl repository](https://en.wikipedia.org/wiki/Public_key_infrastructure) on all major platforms. This tutorial only targets macOS and Linux:

### macOS

```bash
# Download the cfssl and cfssljson binaries
curl -o cfssl https://pkg.cfssl.org/R1.2/cfssl_darwin-amd64
curl -o cfssljson https://pkg.cfssl.org/R1.2/cfssljson_darwin-amd64

# Make them executable
chmod +x cfssl cfssljson

# Install
sudo mv cfssl cfssljson /usr/local/bin/
```

### Linux

```bash
# Download the cfssl and cfssljson binaries
wget -q --show-progress --https-only --timestamping \
  https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 \
  https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64

# Make them executable
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64

# Install
sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl
sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
```

### Windows

To install on Windows, download the following two files:

[https://pkg.cfssl.org/R1.2/cfssl_windows-amd64.exe](https://pkg.cfssl.org/R1.2/cfssl_windows-amd64.exe)

[https://pkg.cfssl.org/R1.2/cfssljson_windows-amd64.exe](https://pkg.cfssl.org/R1.2/cfssljson_windows-amd64.exe)

Then, place them in a folder which is in your system PATH. If you do not have a folder for this, create `C:\bin` and place them there. Then follow these steps:

* Open the Control Panel
* Navigate to the System section
* Select Advanced
* Select Environment Variables
* Select the Path variable and select Edit
* Select New
* Enter C:\bin and save

### Verification

The `cfssl version` command should now output it's version if it is in your path. The `cfssljson` command does not have any output, but if you do not see a message indicating that it is not found, then it should be working.

## kubectl

The `kubectl` command is the official tool for interacting with K8S clusters. It provides a native command line interface for most K8S API functions.

### macOS

```bash
# Download the kubectl binary
curl -o kubectl https://storage.googleapis.com/kubernetes-release/release/v1.10.3/bin/darwin/amd64/kubectl

# Make it executable
chmod +x kubectl

# Install
sudo mv kubectl /usr/local/bin/
```

### Linux

```bash
# Download the kubectl binary
wget https://storage.googleapis.com/kubernetes-release/release/v1.10.3/bin/linux/amd64/kubectl

# Make it executable
chmod +x kubectl

# Install
sudo mv kubectl /usr/local/bin/
```

### Windows

To install on Windows, download the following file:

[https://storage.googleapis.com/kubernetes-release/release/v1.10.3/bin/windows/amd64/kubectl.exe](https://storage.googleapis.com/kubernetes-release/release/v1.10.3/bin/windows/amd64/kubectl.exe)

Then, place them in a folder which is in your system PATH. If you do not have a folder for this, create `C:\bin` and place them there. Then follow these steps:

* Open the Control Panel
* Navigate to the System section
* Select Advanced
* Select Environment Variables
* Select the Path variable and select Edit
* Select New
* Enter C:\bin and save

### Verification

The `kubectl version --client` command will output information about the `kubectl` tool if it is installed.

## Git Bash (WINDOWS ONLY)

If you are using Windows for this tutorial, you will need a UNIX style environment to work with. Specifically, you will need a copy of `bash` for most of the commands to work correctly. An easy to install environment for this is [Git Bash](https://git-scm.com/).

To install Git Bash, download and install the windows version:

[https://git-scm.com/download/win](https://git-scm.com/download/win)

Then, follow the installation instructions.

### Verification

You should now have a "Git Bash" option in the Windows menu. Make sure that you use this for all subsequent sections.

Next: [Provisioning Compute Resources](02-provisioning-compute-resources.md)