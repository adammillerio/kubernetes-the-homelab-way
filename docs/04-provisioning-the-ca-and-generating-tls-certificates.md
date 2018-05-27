# 3: Provisioning the CA and Generating TLS Certificates

## Overview

In this lab, we will be generating the Public Key Infrastructure for our Kubernetes cluster.

Because Kubernetes is a distributed system, there must be some way that communications between nodes can be properly secured. After all, the cluster is on a flat network, so nodes can reach each other freely. What stops a rogue node from registering itself with the cluster? The answer to this question is Public Key Infrastructure, or PKI.

At it's most basic, the PKI setup is a collection of certificates and keys. The certificate is a file which is presented by each node in the cluster, while the key is known only to the node which the key belongs to. Using this key the node is able to "prove" that it is the node that the presented certificate says it is. This will be explored in greater detail on a case-by-case basis as we move along.

## Certificate Authority

The first building block in our cluster's PKI that we will create is the Certificate Authority (CA). This is the master certificate and key which will be used to generate all subsequent certificates. These certificates will contain both the data for the subject of the certificate, such as a worker or master node, as well as the data for the CA which is responsible for verifying the certificate's validity.

The tools we are using to generate these certificates, `cfssl` and `cfssljson`, require a configuration file as well as a Certificate Signing Request (CSR).

First, we will create the CA configuration file for `cfssl`:

```bash
cat > ca-config.json <<EOF
{  
  "signing":{  
    "default":{  
      "expiry":"8760h"
    },
    "profiles":{  
      "kubernetes":{  
        "usages":[  
          "signing",
          "key encipherment",
          "server auth",
          "client auth"
        ],
        "expiry":"8760h"
      }
    }
  }
}
EOF
```

This will create `ca-config.json` which will be referenced frequently during subsequent certificate generation. It specifies the kubernetes profile, which is a suite of tools designed for maintaining the PKI of a Kubernetes cluster. In addition, it specifies that the generated CA certificate will expire 8760 hours (1 year) after it is generated.

Now, we will generate the CSR:

```bash
cat > ca-csr.json <<EOF
{  
  "CN":"Kubernetes",
  "key":{  
    "algo":"rsa",
    "size":2048
  },
  "names":[  
    {  
      "O":"Kubernetes",
      "OU":"CA",
      "C":"${COUNTRY}",
      "L":"${LOCALITY}",
      "ST":"${STATE}"
    }
  ]
}
EOF
```

This creates `ca-csr.json` which is the CSR that we will use to generate the CA certificate. A CSR is exactly what it sounds like. A request to generate a certificate, along with all relevant values. In this example, we specify the RSA algorithm for the key, with a size of 2048 bits. In addition, we provide metadata for the certificate, such as the Organization (O), Organizational Unit (OU), and all information relating to the locality of the cluster.

Using these files, we will now generate the CA certificate and key:

```bash
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

### Verification

After running the command, you should get output similar to the following:

```bash
> cfssl gencert -initca ca-csr.json | cfssljson -bare ca
2018/05/24 21:46:40 [INFO] generating a new CA key and certificate from CSR
2018/05/24 21:46:40 [INFO] generate received request
2018/05/24 21:46:40 [INFO] received CSR
2018/05/24 21:46:40 [INFO] generating key: rsa-2048
2018/05/24 21:46:40 [INFO] encoded CSR
2018/05/24 21:46:40 [INFO] signed certificate with serial number 370640243432406175463470642891371079460982563388
```

You should now have three new files in your working directory, the CSR, CA certificate, and key:

```
ca.csr
ca-key.pem
ca.pem
```

## Admin Certificate

The next certificate we will generate is the "admin" certificate. This will be used by the `kubectl` command to administrate our cluster.

First, we create the CSR:

```bash
cat > admin-csr.json <<EOF
{  
  "CN":"admin",
  "key":{  
    "algo":"rsa",
    "size":2048
  },
  "names":[  
    {  
      "O":"system:masters",
      "OU":"k8s master admin",
      "C":"${COUNTRY}",
      "L":"${LOCALITY}",
      "ST":"${STATE}"
    }
  ]
}
EOF
```

The only elements that have changed are the Organization and Organizational Unit to reflect their purpose.

Then, generate the certificate:

```bash
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
```

### Verification

After running the command, you should get output similar to the following:

```bash
2018/05/24 21:52:41 [INFO] generate received request
2018/05/24 21:52:41 [INFO] received CSR
2018/05/24 21:52:41 [INFO] generating key: rsa-2048
2018/05/24 21:52:42 [INFO] encoded CSR
2018/05/24 21:52:42 [INFO] signed certificate with serial number 190150382662131629228456444028401174922033531326
2018/05/24 21:52:42 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```

The warning is informing us that we do not specify any hostnames in our certificate. If this were a web server serving pages to a browser, this would be unsuitable. However, it is fine for our use case within Kubernetes.

You should now have three new files in your working directory, the CSR, admin certificate, and key:

```
ca.csr
ca-key.pem
ca.pem
```

## Client Certificates

The kubelet process running on each node in the cluster requires a client certificate in order to authenticate with the Kubernetes API server. This process is called the [Node Authorizer](https://kubernetes.io/docs/admin/authorization/node/) which uses a node's client certificate to verify that it is in the "system:nodes" group within the cluster. This will be explained in greater detail when discussing RBAC later in the tutorial.

First, we will generate each master node certificate CSR:

```bash
for i in $(seq 1 ${MASTER_NODE_COUNT}); do
cat > master${i}-csr.json <<EOF
{  
  "CN":"system:node:master${i}",
  "key":{  
    "algo":"rsa",
    "size":2048
  },
  "names":[  
    {  
      "C":"${COUNTRY}",
      "L":"${LOCALITY}",
      "O":"system:nodes",
      "OU":"master${i}",
      "ST":"${STATE}"
    }
  ]
}
EOF
done
```

As well as each worker node certificate CSR:

```bash
for i in $(seq 1 ${WORKER_NODE_COUNT}); do
cat > worker${i}-csr.json <<EOF
{  
  "CN":"system:node:worker${i}",
  "key":{  
    "algo":"rsa",
    "size":2048
  },
  "names":[  
    {  
      "C":"${COUNTRY}",
      "L":"${LOCALITY}",
      "O":"system:nodes",
      "OU":"worker${i}",
      "ST":"${STATE}"
    }
  ]
}
EOF
done
```

Finally, generate the master certificates:

```bash
for i in $(seq 1 ${MASTER_NODE_COUNT}); do
  cfssl gencert \
    -ca=ca.pem \
    -ca-key=ca-key.pem \
    -config=ca-config.json \
    -hostname=master${i}.local \
    -profile=kubernetes \
    master${i}-csr.json | cfssljson -bare master${i}
done
```

And the worker certificates:

```bash
for i in $(seq 1 ${WORKER_NODE_COUNT}); do
  cfssl gencert \
    -ca=ca.pem \
    -ca-key=ca-key.pem \
    -config=ca-config.json \
    -hostname=worker${i}.local \
    -profile=kubernetes \
    worker${i}-csr.json | cfssljson -bare worker${i}
done
```

It is important to note that in these certificates, we specify a hostname. This ensures that the certificate for a node such as `master1.local` can only be used to authenticate a request made to `master1.local`. If instead the certificate for `worker1.local` was presented, the request will fail. This provides an authentication layer for nodes in our cluster.

### Verification

After running the command, you should see similar output for each master node:

```bash
2018/05/24 22:11:19 [INFO] generate received request
2018/05/24 22:11:19 [INFO] received CSR
2018/05/24 22:11:19 [INFO] generating key: rsa-2048
2018/05/24 22:11:19 [INFO] encoded CSR
2018/05/24 22:11:19 [INFO] signed certificate with serial number 156595369515929326447316517702031504720121808313
```

And for each master node, you should have a corresponding CSR, certificate, and key:

```
master1.csr
master1-key.pem
master1.pem
```

As well as similar files for our workers:

```
worker1.csr
worker1-key.pem
worker1.pem
```

## API Server Certificate

The Kubernetes API server is presented upon any request made to the Kubernetes API. This allows for any clients to be able to verify that they are indeed talking to the API server that they expect.

First, we create the CSR:

```bash
cat > kubernetes-csr.json <<EOF
{  
  "CN":"kubernetes",
  "key":{  
    "algo":"rsa",
    "size":2048
  },
  "names":[  
    {  
      "O":"Kubernetes",
      "OU":"API Server",
      "C":"${COUNTRY}",
      "L":"${LOCALITY}",
      "ST":"${STATE}"
    }
  ]
}
EOF
```

Then, generate the certificate:

```bash
for i in $(seq 1 ${MASTER_NODE_COUNT}); do
  if [[ ${i} == 1 ]]; then
    export MASTER_LIST="master1.local"
  else
    export MASTER_LIST="${MASTER_LIST},master${i}.local"
  fi
done

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${MASTER_LIST},127.0.0.1,kubernetes.default \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

The loop at the beginning is to create a comma-separated list of master node hostnames. In addition, the `kubernetes.default` host is provided as well. This is the hostname of the Kubernetes "Service" corresponding to the API server. It is the address that any application running within the cluster will use to reach the API. This is then passed into the certificate generation.

### Verification

After running the command, you should see similar output to below:

```bash
2018/05/24 22:39:10 [INFO] generate received request
2018/05/24 22:39:10 [INFO] received CSR
2018/05/24 22:39:10 [INFO] generating key: rsa-2048
2018/05/24 22:39:10 [INFO] encoded CSR
2018/05/24 22:39:10 [INFO] signed certificate with serial number 624796334096814253613054813979347798112444724954
```

You should now have three new files in your working directory, the CSR, API server certificate, and key:

```
kubernetes.csr
kubernetes-key.pem
kubernetes.pem
```

## Controller Manager Certificate

Each service within the Kubernetes control plane relies on the API server to perform their functions. To do this, they require a special client certificate of their own in order to authenticate. We will now generate the certificate for the Controller Manager:

CSR:

```bash
cat > kube-controller-manager-csr.json <<EOF
{
  "CN":"system:kube-controller-manager",
  "key": {
    "algo":"rsa",
    "size":2048
  },
  "names": [
    {
      "O":"system:kube-controller-manager",
      "OU":"Controller Manager",
      "C":"${COUNTRY}",
      "L":"${LOCALITY}",
      "ST":"${STATE}"
    }
  ]
}
EOF
```

Certificate:

```bash
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```

### Verification

After running the command, you should see similar output to below:

```bash
2018/05/24 22:47:46 [INFO] generate received request
2018/05/24 22:47:46 [INFO] received CSR
2018/05/24 22:47:46 [INFO] generating key: rsa-2048
2018/05/24 22:47:47 [INFO] encoded CSR
2018/05/24 22:47:47 [INFO] signed certificate with serial number 419885037423404141926351068919920688161089046808
2018/05/24 22:47:47 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```

You should now have three new files in your working directory, the CSR, controller manager certificate, and key:

```
kube-controller-manager.csr
kube-controller-manager-key.pem
kube-controller-manager.pem
```

## Scheduler Client Certificate

The other control plane component we will generate a certificate for is the Scheduler.

CSR:

```bash
cat > kube-scheduler-csr.json <<EOF
{
  "CN":"system:kube-scheduler",
  "key": {
    "algo":"rsa",
    "size":2048
  },
  "names": [
    {
      "O":"system:kube-scheduler",
      "OU":"Scheduler",
      "C":"${COUNTRY}",
      "L":"${LOCALITY}",
      "ST":"${STATE}"
    }
  ]
}
EOF
```

Certificate:

```bash
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```

### Verification

After running the command, you should see similar output to below:

```bash
2018/05/24 22:50:13 [INFO] generate received request
2018/05/24 22:50:13 [INFO] received CSR
2018/05/24 22:50:13 [INFO] generating key: rsa-2048
2018/05/24 22:50:13 [INFO] encoded CSR
2018/05/24 22:50:13 [INFO] signed certificate with serial number 699295481514887113613087623112495797175994167607
2018/05/24 22:50:13 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```

You should now have three new files in your working directory, the CSR, scheduler certificate, and key:

```
kube-scheduler.csr
kube-scheduler-key.pem
kube-scheduler.pem
```

## Kube Proxy Certificate

The `kube-proxy` is a daemon which runs on every node in the cluster, and is responsible for the dynamic routing of requests made to Kubernetes Service objects. This will be explained in greater detail in later sections. Like the control plane, it requires a certificate to authenticate and interact with the API server.

CSR:

```bash
cat > kube-proxy-csr.json <<EOF
{  
  "CN":"system:kube-proxy",
  "key":{  
    "algo":"rsa",
    "size":2048
  },
  "names":[  
    {  
      "O":"system:node-proxier",
      "OU":"kube-proxy",
      "C":"${COUNTRY}",
      "L":"${LOCALITY}",
      "ST":"${STATE}"
    }
  ]
}
EOF
```

Certificate:

```bash
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

### Verification

After running the command, you should see similar output to below:

```bash
2018/05/24 22:54:50 [INFO] generate received request
2018/05/24 22:54:50 [INFO] received CSR
2018/05/24 22:54:50 [INFO] generating key: rsa-2048
2018/05/24 22:54:50 [INFO] encoded CSR
2018/05/24 22:54:50 [INFO] signed certificate with serial number 124947576719254086100453305811656660765291800716
2018/05/24 22:54:50 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```

You should now have three new files in your working directory, the CSR, scheduler certificate, and key:

```
kube-proxy.csr
kube-proxy-key.pem
kube-proxy.pem
```

## Service Account Certificate

One of the functions of the Controller Manager is to generate access tokens for Kubernetes ServiceAccount objects. It utilizes a certificate to generate these:

CSR:

```bash
cat > service-account-csr.json <<EOF
{
  "CN":"service-accounts",
  "key": {
    "algo":"rsa",
    "size":2048
  },
  "names": [
    {
      "O": "Kubernetes",
      "OU": "Service Accounts",
      "C":"${COUNTRY}",
      "L":"${LOCALITY}",
      "ST":"${STATE}"
    }
  ]
}
EOF
```

Certificate:

```bash
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account
```

### Verification

After running the command, you should see similar output to below:

```bash
2018/05/24 22:57:47 [INFO] generate received request
2018/05/24 22:57:47 [INFO] received CSR
2018/05/24 22:57:47 [INFO] generating key: rsa-2048
2018/05/24 22:57:47 [INFO] encoded CSR
2018/05/24 22:57:47 [INFO] signed certificate with serial number 562728501647545972247416177408580284981433370833
2018/05/24 22:57:47 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").

```

You should now have three new files in your working directory, the CSR, service account certificate, and key:

```
service-account.csr
service-account-key.pem
service-account.pem
```

Next: [Generating the Data Encryption Config and Key](05-generating-the-data-encryption-config-and-key.md)
