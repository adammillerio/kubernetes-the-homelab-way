# 5: Generating the Data Encryption Config and Key

## Overview

The Kubernetes cluster uses `etcd` to store cluster information. This can include sensitive details such as application passwords and other configuration. Because of this, we will be leveraging Kubernetes' ability to encrypt this cluster data at rest.

## Encryption Key

First, generate the encryption key:

```bash
export ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

This grabs the first 32 characters from the system's pseudo-random number generator `/dev/urandom`. Then, to ensure the key is an ASCII string, we encode this data using Base64.

Now, we generate the encryption configuration file:

```bash
cat > encryption-config.yaml <<EOF
apiVersion: v1
kind: EncryptionConfig
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

Finally, we copy this to each master node:

```bash
for i in $(seq 1 ${MASTER_NODE_COUNT}); do
  vagrant scp encryption-config.yml master${i}:~/
done
```

Next: [Installing the Docker Container Runtime Interface (CRI)](06-installing-the-docker-container-runtime-interface.md)