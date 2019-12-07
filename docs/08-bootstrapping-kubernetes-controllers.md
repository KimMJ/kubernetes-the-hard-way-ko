# Bootstrapping the Kubernetes Control Plane

이 랩에서는 세개의 compute instance들에 걸처 고가용성을 고려하여 설정하며 쿠버네티스 control plane을 부트스트랩 할 것입니다. 또한 외부 load balancer를 생성하여 쿠버네티스 API 서버를 remote client에 대해 노출시킬 수 있습니다. 다음의 컴포넌트들은 각 노드에서 설치될 것입니다: 쿠버네티스 API 서버, Scheduler, Controller Manager

## Prerequisites

이 랩에서의 명령어들은 반드시 각 controller instance에서 동작되어야 합니다: `controller-0`, `controller-1`, `controller-2`. 각 controller instance에 대해 `gcloud` 명령어를 사용하여 로그인하십시오. 예를 들어:

```bash
gcloud compute ssh controller-0
```

### tmux를 통해 명령어를 동시에 수행하기

[tmux](https://github.com/tmux/tmux/wiki)는 여러 compute instance들에 대해 동시에 명령어를 실행시킬 때 사용할 수 있습니다. Prerequisites 랩에서 [Running commands in parallel with tmux](01-prerequisites.md#running-commands-in-parallel-with-tmux) 섹션을 확인하세요. 

## Provision the Kubernetes Control Plane

쿠버네티스 configuration 디렉토리를 생성합니다:

```bash
sudo mkdir -p /etc/kubernetes/config
```

### 쿠버네티스 Controller 바이너리를 다운로드 받고 설치하기

공식 쿠버네티스 릴리즈 바이너리를 다운로드합니다:

```bash
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kubectl"
```

쿠버네티스 바이너리를 설치합니다:

```bash
{
  chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
  sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
}
```

### 쿠버네티스 API 서버를 설정하기

```bash
{
  sudo mkdir -p /var/lib/kubernetes/

  sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem \
    encryption-config.yaml /var/lib/kubernetes/
}
```

instance 내부 IP 주소는 API Server를 클러스터 내부의 멤버들에게 알리는 데 사용됩니다. 현재 compute instance의 내부 IP 주소를 가져옵니다.

```bash
INTERNAL_IP=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/ip)
```

`kube-apiserver.service` systemd unit file을 생성합니다:

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://10.240.0.10:2379,https://10.240.0.11:2379,https://10.240.0.12:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config=api/all \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Kubernetes Controller Manager 설정하기

`kube-controller-manager` kubeconfig를 위치로 옮깁니다:

```bash
sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

`kube-controller-manager.service` systemd unit file을 생성합니다:

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 쿠버네티스 Scheduler 설정하기

`kube-scheduler` kubeconfig를 위치로 옮깁니다:

```bash
sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```

`kube-scheduler.yaml` 설정 파일을 생성합니다:

```bash
cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1alpha1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```

`kube-scheduler.service` systemd unit file을 생성합니다:

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Controller Service들 시작하기

```
{
  sudo systemctl daemon-reload
  sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
  sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
}
```

> 쿠버네티스 API 서버를 완전히 초기화하는 데 최대 10초정도 걸립니다.

### HTTP Health Checks 활성화

[Google Network Load Balancer](https://cloud.google.com/compute/docs/load-balancing/network)는 트래픽을 세 API 서버들로 분배할 때 사용되고 각 API 서버가 TLS connections을 끝내고 client certificates를 검증할 수 있도록 합니다. 네트워크 load balancer는 HTTP health check만 지원합니다. 이는 API 서버에 의해 노출되는 HTTPS 엔드포인트는 사용될 수 없음을 의미합니다. nginx 웹서버를 HTTP health check의 프록시로 쓸 수도 있습니다. 이 섹션에서는 nginx가 설치되어 `80` 포드로 오는 HTTP health check를 받고 연결을 `https://127.0.0.1:6443/healthz`의 API 서버로 프록싱합니다.

> `/healthz` API 서버 엔드포인트는 기본적으로 인증이 필요하지 않습니다.

HTTP health check를 다루는 기본 웹서버를 설치합니다:

```bash
sudo apt-get update
sudo apt-get install -y nginx
```

```bash
cat > kubernetes.default.svc.cluster.local <<EOF
server {
  listen      80;
  server_name kubernetes.default.svc.cluster.local;

  location /healthz {
     proxy_pass                    https://127.0.0.1:6443/healthz;
     proxy_ssl_trusted_certificate /var/lib/kubernetes/ca.pem;
  }
}
EOF
```

```bash
{
  sudo mv kubernetes.default.svc.cluster.local \
    /etc/nginx/sites-available/kubernetes.default.svc.cluster.local

  sudo ln -s /etc/nginx/sites-available/kubernetes.default.svc.cluster.local /etc/nginx/sites-enabled/
}
```

```bash
sudo systemctl restart nginx
```

```bash
sudo systemctl enable nginx
```

### 검증

```bash
kubectl get componentstatuses --kubeconfig admin.kubeconfig
```

```
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-2               Healthy   {"health": "true"}
etcd-0               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
```

nginx HTTP health check 프록시를 테스트합니다:

```bash
curl -H "Host: kubernetes.default.svc.cluster.local" -i http://127.0.0.1/healthz
```

```bash
HTTP/1.1 200 OK
Server: nginx/1.14.0 (Ubuntu)
Date: Sat, 14 Sep 2019 18:34:11 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 2
Connection: keep-alive
X-Content-Type-Options: nosniff

ok
```

> 위의 명령어들은 각 controller node에서 실행되어야 함을 기억하시기 바랍니다: `controller-0`, `controller-1`, `controller-2`

## Kubelet Authorization을 위한 RBAC

이번 섹션에서는 RBAC 권한을 설정하여 쿠버네티스 API 서버가 각 워커 노드에서의 Kubelet API에 접속할 수 있도록 합니다. Kubelet API에 접속하는 것은 메트릭, 로그를 얻을 때, 파드 안에서 명령어를 실행할 때 필요합니다.

> 이 튜토리얼은 Kubelet `--authorization-mode` 플래그를 `Webhook`으로 설정합니다. Webhook 모드는 [SubjectAccessReview](https://kubernetes.io/docs/admin/authorization/#checking-api-access) API를 사용하여 인증을 합니다.

이 섹션에서의 명령어는 전체 클러스터에 영향을 주어 하나의 controller node에서 한번 실행하기만 하면 됩니다.

```bash
gcloud compute ssh controller-0
```

Kubelet API에 접속하고 파드를 관리하는 것과 관련된 대부분의 일반적인 task들을 실행할 수 있는 `system:kube-apiserver-to-kubelet` [ClusterRole](https://kubernetes.io/docs/admin/authorization/rbac/#role-and-clusterrole)을 생성합니다:

```bash
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```

쿠버네티스 API 서버는 `--kubelet-client-certificate` 플래그를 통해 정의된 client certificate를 사용하여 `kubernetes` 유저로 Kubelet에 인증합니다.

`system:kube-apiserver-to-kubelet` ClusterRole을 `kubernetes` 유저로 바인딩 합니다:

```bash
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

## The Kubernetes Frontend Load Balancer

이 섹션에서는 외부 load balancer를 쿠버네티스 API 서버 앞단에 설정합니다. `kubernetes-the-hard-way` 정적 IP 주소는 load balancer에 적용됩니다.

> 이 튜토리얼에서 생성된 compute instance들은 이 섹션을 완료할 수 있는 권한을 가지고 있지 않습니다. **compute instance들을 생성하는 데 사용하는 동일한 장비에서 다음의 명령어를 실행시킵니다**.

### Provision a Network Load Balancer

외부 load balancer network resource들을 생성합니다:

```bash
{
  KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
    --region $(gcloud config get-value compute/region) \
    --format 'value(address)')

  gcloud compute http-health-checks create kubernetes \
    --description "Kubernetes Health Check" \
    --host "kubernetes.default.svc.cluster.local" \
    --request-path "/healthz"

  gcloud compute firewall-rules create kubernetes-the-hard-way-allow-health-check \
    --network kubernetes-the-hard-way \
    --source-ranges 209.85.152.0/22,209.85.204.0/22,35.191.0.0/16 \
    --allow tcp

  gcloud compute target-pools create kubernetes-target-pool \
    --http-health-check kubernetes

  gcloud compute target-pools add-instances kubernetes-target-pool \
   --instances controller-0,controller-1,controller-2

  gcloud compute forwarding-rules create kubernetes-forwarding-rule \
    --address ${KUBERNETES_PUBLIC_ADDRESS} \
    --ports 6443 \
    --region $(gcloud config get-value compute/region) \
    --target-pool kubernetes-target-pool
}
```

### 검증

> 이 튜토리얼에서 생성된 compute instance들은 이 섹션을 완료하는 데 필요한 권한을 가지고 있지 않습니다. **compute instance들을 생성하는 데 사용한 것과 동일한 장비에서 다음의 명령어를 실행하십시오**.

`kubernetes-the-hard-way` 정적 IP를 불러옵니다:

```bash
KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')
```

쿠버네티스 버전 정보에 대한 HTTP request를 생성합니다:

```bash
curl --cacert ca.pem https://${KUBERNETES_PUBLIC_ADDRESS}:6443/version
```

> 결과

```
{
  "major": "1",
  "minor": "15",
  "gitVersion": "v1.15.3",
  "gitCommit": "2d3c76f9091b6bec110a5e63777c332469e0cba2",
  "gitTreeState": "clean",
  "buildDate": "2019-08-19T11:05:50Z",
  "goVersion": "go1.12.9",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

다음: [Bootstrapping the Kubernetes Worker Nodes](09-bootstrapping-kubernetes-workers.md)
