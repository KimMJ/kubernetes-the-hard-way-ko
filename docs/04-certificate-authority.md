# CA 설정 및 TLS Certificates 생성

이 랩에서는 CloudFare의 PKI toolkit인  [cfssl](https://github.com/cloudflare/cfssl)을 통해  [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure)을 설정하고 이를 Certificate Authority의 부트스트랩과 다음의 컴포넌트에 대한 TLS certificates를 생성에 사용합니다: etcd, kube-apiserver, kube-controller-manager, kube-scheduler, kubelet, kube-proxy

## Certificate Authority

이 섹션에는 추가적인 TLS certificates를 생성할 때 사용할 Certificate Authority를 설정합니다.

Generate the CA configuration file, certificate, and private key:

CA 설정 파일, certificate, private key를 생성합니다.

```bash
{

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

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

}
```

결과:

```
ca-key.pem
ca.pem
```

## Client와 Server Certificates

In this section you will generate client and server certificates for each Kubernetes component and a client certificate for the Kubernetes `admin` user.

이 섹션에서는 각 쿠버네티스 컴포넌트와 쿠버네티스 `admin` 유저에 대한 client certificate에 대해 client와 server certificate들을 생성합니다.

### The Admin Client Certificate

`admin` client certificate와 private key를 생성합니다:

```bash
{

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

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin

}
```

결과:

```
admin-key.pem
admin.pem
```

### The Kubelet Client Certificates

쿠버네티스는 Node Authorizer라고 부르는 [특수 목적의 authorization 모드](https://kubernetes.io/docs/admin/authorization/node/)를 사용하고 이는 특히 [Kubelets](https://kubernetes.io/docs/concepts/overview/components/#kubelet)에 의한 API 요청에 권한을 부여하는 데 사용합니다. Node Authorizer에 의해 권한을 부여받기 위해서 Kubelet은 반드시 `system:node:<nodeName>`라는 username을 가지고 `system:nodes` 그룹 안에 있다고 증명할 수 있는 credential을 사용해야 합니다. 이 섹션에서는 각 쿠버네티스 워커 노드들에 대해 Node Authorizer의 요구사항을 충족하는 certificate를 생성할 것입니다.

각 쿠버네티스 워커 노드에 대해 certificate와 private key를 생성합니다:

```bash
for instance in worker-0 worker-1 worker-2; do
cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance}",
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

EXTERNAL_IP=$(gcloud compute instances describe ${instance} \
  --format 'value(networkInterfaces[0].accessConfigs[0].natIP)')

INTERNAL_IP=$(gcloud compute instances describe ${instance} \
  --format 'value(networkInterfaces[0].networkIP)')

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance},${EXTERNAL_IP},${INTERNAL_IP} \
  -profile=kubernetes \
  ${instance}-csr.json | cfssljson -bare ${instance}
done
```

결과:

```
worker-0-key.pem
worker-0.pem
worker-1-key.pem
worker-1.pem
worker-2-key.pem
worker-2.pem
```

### The Controller Manager Client Certificate

`kube-controller-manger` client certificate와 private key를 생성합니다:

```bash
{

cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

}
```

결과:

```
kube-controller-manager-key.pem
kube-controller-manager.pem
```


### The Kube Proxy Client Certificate

`kube-proxy` client certificate와 private key를 생성합니다:

```bash
{

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

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy

}
```

결과:

```
kube-proxy-key.pem
kube-proxy.pem
```

### The Scheduler Client Certificate

`kube-scheduler` client certificate와 private key를 생성합니다:

```bash
{

cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler

}
```

결과:

```
kube-scheduler-key.pem
kube-scheduler.pem
```


### The Kubernetes API Server Certificate

`kubernetes-the-hard-way` 정적 IP 주소는 쿠버네티스 API Server certificate에 대한 subject alternative names(SAN)의 리스트에 포함될 것입니다. 이는 certificate가 remote clients에 의해서 인증될 수 있음을 보장합니다.

쿠버네티스 API Server certificate와 private key를 생성합니다:

```bash
{

KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')

KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local

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

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,10.240.0.10,10.240.0.11,10.240.0.12,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes

}
```

> 쿠버네티스 API Server는 [control plane bootstrapping](08-bootstrapping-kubernetes-controllers.md#configure-the-kubernetes-api-server) 랩을 진행하는 동안 내부 클러스터 서비스를 위해 예약된, 주소 범위(`10.32.0.0/24`)의 첫 IP 주소(`10.32.0.1`)에 연결되는 `kubernetes` 내부 dns name에 자동 할당됩니다. 

결과:

```
kubernetes-key.pem
kubernetes.pem
```

## The Service Account Key Pair

쿠버네티스 Controller Manager는 [managing service accounts](https://kubernetes.io/docs/admin/service-accounts-admin/) 문서에 설명되었듯이 key pair를 이용하여 service account를 생성하고 서명합니다.

`service-account` certificate와 private key를 생성합니다:

```bash
{

cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
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

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account

}
```

결과:

```
service-account-key.pem
service-account.pem
```


## Client와 Server Certificates의 분배

적절한 certificate들과 private key들을 각 worker instance에 복사합니다:

```bash
for instance in worker-0 worker-1 worker-2; do
  gcloud compute scp ca.pem ${instance}-key.pem ${instance}.pem ${instance}:~/
done
```

적절한 certificate들과 private key들을 각 controller instance에 복사합니다:

```bash
for instance in controller-0 controller-1 controller-2; do
  gcloud compute scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem ${instance}:~/
done
```

> The `kube-proxy`, `kube-controller-manager`, `kube-scheduler`, and `kubelet` client certificates will be used to generate client authentication configuration files in the next lab.



> `kube-proxy`, `kube-controller-manager`, `kube-scheduler`, `kubelet` client certificate들은 다음 랩에서 client authentication configuration file들을 생성하는 데 사용됩니다.

다음: [Generating Kubernetes Configuration Files for Authentication](05-kubernetes-configuration-files.md)
