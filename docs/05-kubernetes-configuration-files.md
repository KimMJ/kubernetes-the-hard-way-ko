# Authentication을 위한 Kubernetes Configuration Files 생성하기

이 랩은 쿠버네티스 클라이언트가 쿠버네티스 API 서버를 찾고 인증할 수 있도록 하는  kubeconfigs라고도 알려진 [쿠버네티스 configuration files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)을 생성합니다.

## Client Authentication Configs

이 섹션에서는 `controller manager`, `kubelet`, `kube-proxy`, `scheduler` 클라이언트와 `admin` 유저를 위한 kubeconfig 파일을 생성합니다.

### Kubernetes Public IP Address

각 kubeconfig는 연결할 쿠버네티스 API 서버를 필요로 합니다. 고가용성을 지원하기 위해 쿠버네티스 API 서버 앞단에 있는 외부 load balancer로 할당된 IP 주소가 사용됩니다.

`kubernetes-the-hard-way`의 정적 IP 주소를 얻습니다:

```bash
KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')
```

### The kubelet Kubernetes Configuration File

Kubelet에 대한 kubeconfig 파일을 생성할 때 반드시 Kubelet의 노드 이름과 매칭이 되는 client certificate를 사용해야 합니다. 이는 Kubelet이 쿠버네티스 [Node Authorizer](https://kubernetes.io/docs/admin/authorization/node/)에 의해 적절하게 권한을 받도록 합니다.

> 다음의 명령어는 반드시 [Generating TLS Certificates](04-certificate-authority.md) 랩을 진행하던 중 SSL certificate를 생성할 때 사용했던 디렉토리와 같은 디렉토리에서 실행되어야 합니다.

각 워커 노드에 대해 kubeconfig 파일을 생성합니다:

```bash
for instance in worker-0 worker-1 worker-2; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```

결과:

```
worker-0.kubeconfig
worker-1.kubeconfig
worker-2.kubeconfig
```

### The kube-proxy Kubernetes Configuration File

`kube-proxy` 서비스에 대한 kubeconfig 파일을 생성합니다:

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.pem \
    --client-key=kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
}
```

결과:

```
kube-proxy.kubeconfig
```

### The kube-controller-manager Kubernetes Configuration File

`kube-controller-manager` 서비스에 대한 kubeconfig 파일을 생성합니다:

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
}
```

결과:

```
kube-controller-manager.kubeconfig
```


### The kube-scheduler Kubernetes Configuration File

`kube-scheduler` 서비스에 대한 kubeconfig 파일을 생성합니다:

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
}
```

Results:

```
kube-scheduler.kubeconfig
```

### The admin Kubernetes Configuration File

`admin` 유저에 대한 kubeconfig 파일을 생성합니다:

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default --kubeconfig=admin.kubeconfig
}
```

결과:

```
admin.kubeconfig
```

## Kubernetes Configuration File들의 분배

적절한 `kubelet`과 `kube-proxy` kubeconfig 파일들을 각 worker instance에 복사합니다.

```bash
for instance in worker-0 worker-1 worker-2; do
  gcloud compute scp ${instance}.kubeconfig kube-proxy.kubeconfig ${instance}:~/
done
```

적절한 `kube-controller-manger`와 `kube-scheduler` kubeconfig 파일들을 각 controller instance에 복사합니다.

```bash
for instance in controller-0 controller-1 controller-2; do
  gcloud compute scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig ${instance}:~/
done
```

다음: [Generating the Data Encryption Config and Key](06-data-encryption-keys.md)

