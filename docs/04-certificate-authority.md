# Provisioning a CA and Generating TLS Certificates

In this lab you will provision a [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) using CloudFlare's PKI toolkit, [cfssl](https://github.com/cloudflare/cfssl), then use it to bootstrap a Certificate Authority, and generate TLS certificates for the following components: etcd, kube-apiserver, kubelet, and kube-proxy.

## Certificate Authority

In this section you will provision a Certificate Authority that can be used to generate additional TLS certificates.

Create the CA configuration file:

```
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF
```

Create the CA certificate signing request:

```
cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oregon"
    }
  ]
}
EOF
```

Generate the CA certificate and private key:

```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

Results:

```
ca-key.pem
ca.pem
```

## Client and Server Certificates

In this section you will generate client and server certificates for each Kubernetes component and a client certificate for the Kubernetes `admin` user.

### The Admin Client Certificate

Create the `admin` client certificate signing request:

```
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF
```

Generate the `admin` client certificate and private key:

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
```

Results:

```
admin-key.pem
admin.pem
```

### The Kubelet Client Certificates

Kubernetes uses a [special-purpose authorization mode](https://kubernetes.io/docs/admin/authorization/node/) called Node Authorizer, that specifically authorizes API requests made by [Kubelets](https://kubernetes.io/docs/concepts/overview/components/#kubelet). In order to be authorized by the Node Authorizer, Kubelets must use a credential that identifies them as being in the `system:nodes` group, with a username of `system:node:<nodeName>`. In this section you will create a certificate for each Kubernetes worker node that meets the Node Authorizer requirements.

Generate a certificate and private key for each Kubernetes worker node:

```
for instance in ip-10-251-0-20 ip-10-251-4-20 ip-10-251-8-20; do
cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance}.ec2.internal",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF
done

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=ip-10-251-0-20.ec2.internal,ip-10-251-0-20,10.251.0.20 \
  -profile=kubernetes \
  ip-10-251-0-20-csr.json | cfssljson -bare ip-10-251-0-20
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=ip-10-251-4-20.ec2.internal,ip-10-251-4-20,10.251.4.20 \
  -profile=kubernetes \
  ip-10-251-4-20-csr.json | cfssljson -bare ip-10-251-4-20
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=ip-10-251-8-20.ec2.internal,ip-10-251-8-20,10.251.8.20 \
  -profile=kubernetes \
  ip-10-251-8-20-csr.json | cfssljson -bare ip-10-251-8-20
```

Results:

```
ip-10-251-0-20-key.pem
ip-10-251-0-20.pem
ip-10-251-4-20-key.pem
ip-10-251-4-20.pem
ip-10-251-8-20-key.pem
ip-10-251-8-20.pem
```

### The kube-proxy Client Certificate

Create the `kube-proxy` client certificate signing request:

```
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:node-proxier",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF
```

Generate the `kube-proxy` client certificate and private key:

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

Results:

```
kube-proxy-key.pem
kube-proxy.pem
```

### The Kubernetes API Server Certificate

The `kubernetes-the-hard-way` ELB DNS name will be included in the list of subject alternative names for the Kubernetes API Server certificate. This will ensure the certificate can be validated by remote clients.

Retrieve the `kubernetes-the-hard-way` static IP address:

Create the Kubernetes API Server certificate signing request:

```
cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF
```

Generate the Kubernetes API Server certificate and private key:

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.251.252.1,10.251.0.10,10.251.4.10,10.251.8.10,127.0.0.1,kubernetes.default,$elbdns \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

Results:

```
kubernetes-key.pem
kubernetes.pem
```

## Distribute the Client and Server Certificates

Copy the appropriate certificates and private keys to each worker instance:

```
scp ca.pem ip-10-251-0-20-key.pem ip-10-251-0-20.pem 10.251.0.20:~/
scp ca.pem ip-10-251-4-20-key.pem ip-10-251-4-20.pem 10.251.4.20:~/
scp ca.pem ip-10-251-8-20-key.pem ip-10-251-8-20.pem 10.251.8.20:~/
```

Copy the appropriate certificates and private keys to each controller instance:

```
scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem 10.251.0.10:~/
scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem 10.251.4.10:~/
scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem 10.251.8.10:~/
```

> The `kube-proxy` and `kubelet` client certificates will be used to generate client authentication configuration files in the next lab.

Next: [Generating Kubernetes Configuration Files for Authentication](05-kubernetes-configuration-files.md)
