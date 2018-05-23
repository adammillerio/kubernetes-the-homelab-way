# Installing the Client Tools

In this lab, we will be installing the client tools that will be necessary to bootstrap the K8S cluster.

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
wget https://storage.googleapis.com/kubernetes-release/release/v1.10.2/bin/linux/amd64/kubectl

# Make it executable
chmod +x kubectl

# Install
sudo mv kubectl /usr/local/bin/
```

### Verification

The `kubectl version --client` command will output information about the `kubectl` tool if it is installed.

Next: [Provisioning the CA and Generating TLS Certificates](03-provisioning-the-ca-and-generating-tls-certificates.md)