# Deploying the DNS Cluster Add-on

이 랩에서는 [CoreDNS](https://coredns.io/)을 통한 service discovery를 기반으로 DNS를 제공해주는 [DNS add-on](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)을 쿠버네티스 클러스터 내에 동작하는 어플리케이션에 배포합니다.

## The DNS Cluster Add-on

`coredns` 클러스터 add-on을 배포합니다:

```bash
kubectl apply -f https://storage.googleapis.com/kubernetes-the-hard-way/coredns.yaml
```

> 결과

```
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.extensions/coredns created
service/kube-dns created
```

`kube-dns` deployment에 의해 생성된 파드의 리스트를 확인합니다:

```bash
kubectl get pods -l k8s-app=kube-dns -n kube-system
```

> 결과

```
NAME                       READY   STATUS    RESTARTS   AGE
coredns-699f8ddd77-94qv9   1/1     Running   0          20s
coredns-699f8ddd77-gtcgb   1/1     Running   0          20s
```

## Verification

`busybox` deployment를 생성합니다:

```bash
kubectl run --generator=run-pod/v1 busybox --image=busybox:1.28 --command -- sleep 3600
```

`busybox` deployment에 의해 생성된 파드의 리스트를 확인합니다:

```bash
kubectl get pods -l run=busybox
```

> 결과

```
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          3s
```

`busybox` 파드의 전체 이름을 불러옵니다:

```bash
POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
```

`busybox` 파드 내부에서 `kubernetes` 서비스에 대해 DNS lookup을 실행합니다:

```bash
kubectl exec -ti $POD_NAME -- nslookup kubernetes
```

> 결과

```
Server:    10.32.0.10
Address 1: 10.32.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.32.0.1 kubernetes.default.svc.cluster.local
```

다음: [Smoke Test](13-smoke-test.md)
