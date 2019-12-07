# Smoke Test

이 랩에서는 몇가지 task를 진행하여 당신의 쿠버네티스 클러스터가 올바른 기능을 하고 있는지 확인할 것입니다.

## Data Encryption

이 섹션에서는 [encrypt secret data at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#verifying-that-data-is-encrypted)가 정상적으로 동작하는지 확인할 것입니다.

generic 시크릿을 생성합니다:

```bash
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
```

etcd 안에 저장되어 있는 `kubernetes-the-hard-way` 시크릿의 hexdump를 출력합니다:

```bash
gcloud compute ssh controller-0 \
  --command "sudo ETCDCTL_API=3 etcdctl get \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem\
  /registry/secrets/default/kubernetes-the-hard-way | hexdump -C"
```

> 결과

```
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |etes-the-hard-wa|
00000030  79 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |y.k8s:enc:aescbc|
00000040  3a 76 31 3a 6b 65 79 31  3a 44 ac 6e ac 11 2f 28  |:v1:key1:D.n../(|
00000050  02 46 3d ad 9d cd 68 be  e4 cc 63 ae 13 e4 99 e8  |.F=...h...c.....|
00000060  6e 55 a0 fd 9d 33 7a b1  17 6b 20 19 23 dc 3e 67  |nU...3z..k .#.>g|
00000070  c9 6c 47 fa 78 8b 4d 28  cd d1 71 25 e9 29 ec 88  |.lG.x.M(..q%.)..|
00000080  7f c9 76 b6 31 63 6e ea  ac c5 e4 2f 32 d7 a6 94  |..v.1cn..../2...|
00000090  3c 3d 97 29 40 5a ee e1  ef d6 b2 17 01 75 a4 a3  |<=.)@Z.......u..|
000000a0  e2 c2 70 5b 77 1a 0b ec  71 c3 87 7a 1f 68 73 03  |..p[w...q..z.hs.|
000000b0  67 70 5e ba 5e 65 ff 6f  0c 40 5a f9 2a bd d6 0e  |gp^.^e.o.@Z.*...|
000000c0  44 8d 62 21 1a 30 4f 43  b8 03 69 52 c0 b7 2e 16  |D.b!.0OC..iR....|
000000d0  14 a5 91 21 29 fa 6e 03  47 e2 06 25 45 7c 4f 8f  |...!).n.G..%E|O.|
000000e0  6e bb 9d 3b e9 e5 2d 9e  3e 0a                    |n..;..-.>.|
```

etcd key는 반드시 `aescbc` provider가 `key` 암호화 키를 가지고 데이터를 암호화했다는 것을 나타내는 `k8s:enc:aescbc:v1:key1`이라는 접두사가 있어야 합니다.

## Deployments

이 섹션에서는 [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)를 생성하고 관리할 수 있는지 확인할 것입니다.

[nginx](https://nginx.org/en/) 웹 서버에 대한 deployment를 생성합니다:

```bash
kubectl create deployment nginx --image=nginx
```

`nginx` deployment에 의해 생성된 파드들을 확인합니다:

```bash
kubectl get pods -l app=nginx
```

> 결과

```
NAME                     READY   STATUS    RESTARTS   AGE
nginx-554b9c67f9-vt5rn   1/1     Running   0          10s
```

### Port Forwarding

이 섹션에서는 [port forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)을 이용하여 원격에서 어플리케이션에 접속할 수 있는지를 확인할 것입니다:

`nginx` 파드의 전체 이름을 불러옵니다:

```bash
POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath="{.items[0].metadata.name}")
```

로컬 머신의 `8080` 포트를 `nginx` 파드의 `80` 포트로 포워딩합니다:

```bash
kubectl port-forward $POD_NAME 8080:80
```

> 결과

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

새로운 터미널을 열어 포워딩한 주소를 사용하여 HTTP request를 생성합니다:

```bash
curl --head http://127.0.0.1:8080
```

> 결과

```
HTTP/1.1 200 OK
Server: nginx/1.17.3
Date: Sat, 14 Sep 2019 21:10:11 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 13 Aug 2019 08:50:00 GMT
Connection: keep-alive
ETag: "5d5279b8-264"
Accept-Ranges: bytes
```

이전의 터미널로 돌아가서 `nginx` 파드로의 포트 포워딩을 중단하세요:

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
^C
```

### Logs

이번 섹션에서는 [컨테이너 로그](https://kubernetes.io/docs/concepts/cluster-administration/logging/)를 불러올 수 있는지 확인합니다.

`nginx` 파드 로그를 출력합니다:

```bash
kubectl logs $POD_NAME
```

> 결과

```
127.0.0.1 - - [14/Sep/2019:21:10:11 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.52.1" "-"
```

### Exec

이 섹션에서는 [컨테이너 내에서 커맨드를 실행](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/#running-individual-commands-in-a-container)할 수 있는지 확인합니다.

`nginx` 컨테이너 안에서 `nginx -v`를 실행하여 nginx의 버전을 출력합니다:

```bash
kubectl exec -ti $POD_NAME -- nginx -v
```

> 결과

```
nginx version: nginx/1.17.3
```

## Services

이 섹션에서는 [Service](https://kubernetes.io/docs/concepts/services-networking/service/)를 이용하여 어플리케이션을 노출시킬 수 있는지 확인합니다.

[NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) 서비스를 이용하여 `nginx` deployment를 외부로 노출시킵니다.

```bash
kubectl expose deployment nginx --port 80 --type NodePort
```

> LoadBalancer 서비스 타입은 [cloud provider integration](https://kubernetes.io/docs/getting-started-guides/scratch/#cloud-provider)으로 설정되어있지 않기 때문에 사용할 수 없습니다. cloud provider integration으로 설정하는 것은 이 튜토리얼의 범위 밖입니다.

`nginx` 서비스에 할당된 node port를 불러옵니다:

```bash
NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```

`nginx` node port에 원격으로 접속할 수 있는 firewall rule을 생성합니다:

```bash
gcloud compute firewall-rules create kubernetes-the-hard-way-allow-nginx-service \
  --allow=tcp:${NODE_PORT} \
  --network kubernetes-the-hard-way
```

worker instance의 외부 IP 주소를 불러옵니다:

```bash
EXTERNAL_IP=$(gcloud compute instances describe worker-0 \
  --format 'value(networkInterfaces[0].accessConfigs[0].natIP)')
```

외부 IP 주소와 `nginx` node port를 이용하여 HTTP request를 생성합니다:

```bash
curl -I http://${EXTERNAL_IP}:${NODE_PORT}
```

> 결과

```
HTTP/1.1 200 OK
Server: nginx/1.17.3
Date: Sat, 14 Sep 2019 21:12:35 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 13 Aug 2019 08:50:00 GMT
Connection: keep-alive
ETag: "5d5279b8-264"
Accept-Ranges: bytes
```

다음: [Cleaning Up](14-cleanup.md)
