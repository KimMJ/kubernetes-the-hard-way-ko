# Configuring kubectl for Remote Access

이 랩에서는 `admin` 유저 credential들을 기반으로 `kubectl` 명령어 도구를 위한 kubeconfig 파일을 생성할 것입니다.

> 이 랩에서는 명령어들을 admin client certificate들을 생성했을 때와 동일한 디렉토리에서 진행하면 됩니다.

## The Admin Kubernetes Configuration File

각 kubeconfig는 연결할 쿠버네티스 API 서버를 필요로 합니다. 고가용성을 지원하기 위해 쿠버네티스 API 서버 앞단에 있는 외부 load balancer로 할당된 IP 주소를 사용해야합니다.

`admin` 유저로 인증하기에 적합한 kubeconfig 파일을 생성하세요:

```bash
{
  KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
    --region $(gcloud config get-value compute/region) \
    --format 'value(address)')

  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem

  kubectl config set-context kubernetes-the-hard-way \
    --cluster=kubernetes-the-hard-way \
    --user=admin

  kubectl config use-context kubernetes-the-hard-way
}
```

## 검증

원격 쿠버네티스 클러스터의 health를 체크하세요:

```bash
kubectl get componentstatuses
```

> 결과

```
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-1               Healthy   {"health":"true"}
etcd-2               Healthy   {"health":"true"}
etcd-0               Healthy   {"health":"true"}
```

원격 쿠버네티스 클러스터의 노드 리스트를 확인하세요:

```
kubectl get nodes
```

> 결과

```
NAME       STATUS   ROLES    AGE    VERSION
worker-0   Ready    <none>   2m9s   v1.15.3
worker-1   Ready    <none>   2m9s   v1.15.3
worker-2   Ready    <none>   2m9s   v1.15.3
```

다음: [Provisioning Pod Network Routes](11-pod-network-routes.md)
