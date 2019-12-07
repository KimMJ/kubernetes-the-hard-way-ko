# Provisioning Pod Network Routes

파드는 노드의 파드 CIDR range로부터 IP를 할당받아 해당 노드로 스케쥴링 됩니다. 이 때 파드는 네트워크 [라우트](https://cloud.google.com/compute/docs/vpc/routes)가 없기 때문에 다른 노드에서의 다른 파드와 통신할 수 없습니다.

이 랩에서는 각 워커 노드에 대해 노드의 파드 CIDR range를 노드의 내부 IP 주소로 매핑하는 라우트를 생성할 것입니다.

> [다른 방법](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this)을 통해 쿠버네티스의 네트워킹 모델을 구현하는 방법도 있습니다.

## The Routing Table

이 섹션에서는 `kubernetes-the-hard-way` VPC 네트워크에서 라우트를 생성하는 데 필요한 정보들을 수집할 것입니다.

각 worker instance에 대해 내부 IP 주소와 파드 CIDR range를 출력하세요:

```bash
for instance in worker-0 worker-1 worker-2; do
  gcloud compute instances describe ${instance} \
    --format 'value[separator=" "](networkInterfaces[0].networkIP,metadata.items[0].value)'
done
```

> 결과

```
10.240.0.20 10.200.0.0/24
10.240.0.21 10.200.1.0/24
10.240.0.22 10.200.2.0/24
```

## Routes

각 워커 instance에 대해 네트워크 라우트를 생성하세요:

```bash
for i in 0 1 2; do
  gcloud compute routes create kubernetes-route-10-200-${i}-0-24 \
    --network kubernetes-the-hard-way \
    --next-hop-address 10.240.0.2${i} \
    --destination-range 10.200.${i}.0/24
done
```

`kubernetes-the-hard-way` VPC 네트워크에서의 라우트 리스트를 확인합니다:

```bash
gcloud compute routes list --filter "network: kubernetes-the-hard-way"
```

> 결과

```
NAME                            NETWORK                  DEST_RANGE     NEXT_HOP                  PRIORITY
default-route-081879136902de56  kubernetes-the-hard-way  10.240.0.0/24  kubernetes-the-hard-way   1000
default-route-55199a5aa126d7aa  kubernetes-the-hard-way  0.0.0.0/0      default-internet-gateway  1000
kubernetes-route-10-200-0-0-24  kubernetes-the-hard-way  10.200.0.0/24  10.240.0.20               1000
kubernetes-route-10-200-1-0-24  kubernetes-the-hard-way  10.200.1.0/24  10.240.0.21               1000
kubernetes-route-10-200-2-0-24  kubernetes-the-hard-way  10.200.2.0/24  10.240.0.22               1000
```

다음: [Deploying the DNS Cluster Add-on](12-dns-addon.md)
