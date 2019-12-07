# Bootstrapping the etcd Cluster

쿠버네티스 컴포넌트는 stateless이고 클러스터의 상태를 [etcd](https://github.com/etcd-io/etcd)에 저장합니다. 이 랩에서는 세 노드의 etcd 클러스터를 부트스트랩하고 이를 고가용성과 보안 원격 접속을 고려하여 설정합니다.

## Prerequisites

이 랩에서의 커맨드는 반드시 각 controller instance에서 실행되어야 합니다: `controller-0`, `controller-1`, `controller-2`. 각 controller instance에 `gcloud` 명령어를 통해 로그인하십시오. 예를 들어:

```bash
gcloud compute ssh controller-0
```

### tmux를 통해 명령어를 동시에 수행하기

[tmux](https://github.com/tmux/tmux/wiki)는 여러 compute instance들에 대해 동시에 명령어를 실행시킬 때 사용할 수 있습니다. Prerequisites 랩에서 [Running commands in parallel with tmux](01-prerequisites.md#running-commands-in-parallel-with-tmux) 섹션을 확인하세요. 

## Bootstrapping an etcd Cluster Member

### etcd 바이너리 다운로드 및 설치

공식 etcd 릴리즈 바이너리를 [etcd](https://github.com/etcd-io/etcd) GitHub project에서 다운로드 받으세요:

```bash
wget -q --show-progress --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/v3.4.0/etcd-v3.4.0-linux-amd64.tar.gz"
```

`etcd` 서버와 `etcdctl` 명령어 도구를 추출하고 설치합니다:

```bash
{
  tar -xvf etcd-v3.4.0-linux-amd64.tar.gz
  sudo mv etcd-v3.4.0-linux-amd64/etcd* /usr/local/bin/
}
```

### etcd Server 설정

```bash
{
  sudo mkdir -p /etc/etcd /var/lib/etcd
  sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
}
```

instance 내부 IP 주소는 client request을 처리하고 etcd cluster peer들과 통신하는 데 사용됩니다:

```bash
INTERNAL_IP=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/ip)
```

각 etcd 멤버는 반드시 etcd 클러스터 내에서 유일한 이름을 가지고 있어야 합니다. etcd 이름을 현재 compute instance의 hostname과 일치하도록 설정합니다.

```bash
ETCD_NAME=$(hostname -s)
```

`etcd.service` systemd unit file을 생성합니다:

```bash
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://10.240.0.10:2380,controller-1=https://10.240.0.11:2380,controller-2=https://10.240.0.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### etcd Server 시작하기

```bash
{
  sudo systemctl daemon-reload
  sudo systemctl enable etcd
  sudo systemctl start etcd
}
```

> 위의 명령어들은 각 controller node에서 실행되어야 함을 기억합시다: `controller-0`, `controller-1`, `controller-2`

## 검증

etcd 클러스터 멤버 리스트를 확인합니다:

```bash
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```

> 결과

```
3a57933972cb5131, started, controller-2, https://10.240.0.12:2380, https://10.240.0.12:2379
f98dc20bce6225a0, started, controller-0, https://10.240.0.10:2380, https://10.240.0.10:2379
ffed16798470cab5, started, controller-1, https://10.240.0.11:2380, https://10.240.0.11:2379
```

다음: [Bootstrapping the Kubernetes Control Plane](08-bootstrapping-kubernetes-controllers.md)
