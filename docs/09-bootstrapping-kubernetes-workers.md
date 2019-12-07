# Bootstrapping the Kubernetes Worker Nodes

이 랩에서는 세 쿠버네티스 워커 노드를 부트스트랩 할 것입니다. 다음의 컴포넌트들이 각 노드에 설치되어 있어야 합니다: [runc](https://github.com/opencontainers/runc), [container networking plugins](https://github.com/containernetworking/cni), [containerd](https://github.com/containerd/containerd), [kubelet](https://kubernetes.io/docs/admin/kubelet), [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies).

## Prerequisites

이 랩에서의 명령어들은 반드시 각 worker instance에서 실행되어야 합니다: `worker-0`, `worker-1`, `worker-2`. `gcloud` 명령어로 각 worker instance에 로그인합니다. 예를 들어:

```bash
gcloud compute ssh worker-0
```

### tmux를 통해 명령어를 동시에 수행하기

[tmux](https://github.com/tmux/tmux/wiki)는 여러 compute instance들에 대해 동시에 명령어를 실행시킬 때 사용할 수 있습니다. Prerequisites 랩에서 [Running commands in parallel with tmux](01-prerequisites.md#running-commands-in-parallel-with-tmux) 섹션을 확인하세요. 

## Provisioning a Kubernetes Worker Node

OS dependency들을 설치합니다:

```bash
{
  sudo apt-get update
  sudo apt-get -y install socat conntrack ipset
}
```

> socat 바이너리는 `kubectl port-forward` 명령어를 가능하게 만들어줍니다.

### Swap 비활성화

기본적으로 kubelet은 [swap](https://help.ubuntu.com/community/SwapFaq)이 활성화 되어있을 경우 시작이 실패하게 됩니다. 쿠버네티스가 올바른 리소스 할당과 서비스의 퀄리티를 제공하도록 하기 위해 swap을 비활성화 하는것은 [추천](https://github.com/kubernetes/kubernetes/issues/7294) 사항입니다.

swap이 활성화 되어있는지 확인하세요:

```bash
sudo swapon --show
```

결과가 비어있다면 swap은 활성화되지 않은 것입니다. swap이 활성화되어있으면 다음의 명령어를 통해 swap을 즉시 비활성화할 수 있습니다:

```bash
sudo swapoff -a
```

> swap이 재부팅 후에도 off 상태로 유지되도록 하려면 Linux distro 문서를 확인하세요.

### Worker Binaries 다운로드 및 설치

```bash
wget -q --show-progress --https-only --timestamping \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.15.0/crictl-v1.15.0-linux-amd64.tar.gz \
  https://github.com/opencontainers/runc/releases/download/v1.0.0-rc8/runc.amd64 \
  https://github.com/containernetworking/plugins/releases/download/v0.8.2/cni-plugins-linux-amd64-v0.8.2.tgz \
  https://github.com/containerd/containerd/releases/download/v1.2.9/containerd-1.2.9.linux-amd64.tar.gz \
  https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kubelet
```

설치 디렉토리를 생성하세요:

```bash
sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

워커 바이너리들을 설치하세요:

```bash
{
  mkdir containerd
  tar -xvf crictl-v1.15.0-linux-amd64.tar.gz
  tar -xvf containerd-1.2.9.linux-amd64.tar.gz -C containerd
  sudo tar -xvf cni-plugins-linux-amd64-v0.8.2.tgz -C /opt/cni/bin/
  sudo mv runc.amd64 runc
  chmod +x crictl kubectl kube-proxy kubelet runc 
  sudo mv crictl kubectl kube-proxy kubelet runc /usr/local/bin/
  sudo mv containerd/bin/* /bin/
}
```

### CNI 네트워킹 설정하기

현재 compute instance에 대한 파드 CIDR range를 불러오세요:

```bash
POD_CIDR=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/attributes/pod-cidr)
```

`bridge` 네트워크 설정 파일을 생성하세요:

```bash
cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF
```

`loopback` 네트워크 설정 파일을 생성하세요:

```bash
cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
{
    "cniVersion": "0.3.1",
    "name": "lo",
    "type": "loopback"
}
EOF
```

### containerd 설정하기

`containerd` 설정 파일을 생성하세요:

```bash
sudo mkdir -p /etc/containerd/
```

```bash
cat << EOF | sudo tee /etc/containerd/config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = ""
EOF
```

`containerd.service` systemd unit 파일을 생성하세요:

```bash
cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF
```

### Kubelet 설정하기

```bash
{
  sudo mv ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/
  sudo mv ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
  sudo mv ca.pem /var/lib/kubernetes/
}
```

`kubelet-config.yaml` 설정 파일을 생성하세요:

```bash
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${HOSTNAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${HOSTNAME}-key.pem"
EOF
```

> `resolvConf` 설정은 `systemd-resolved`를 동작시키는 시스템에서 CoreDNS를 service discovery로 사용할 때 loop들을 피하기 위하여 사용됩니다.

`kubelet.service` systemd unit 파일을 생성하세요:

```bash
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Kubernetes Proxy 설정하기

```bash
sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

`kube-proxy-config.yaml` 설정 파일을 생성하세요:

```bash
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF
```

`kube-proxy.service` systemd unit 파일을 생성하세요:

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Worker Services 시작하기

```bash
{
  sudo systemctl daemon-reload
  sudo systemctl enable containerd kubelet kube-proxy
  sudo systemctl start containerd kubelet kube-proxy
}
```

> 위의 명령어들을 각 워커 노드들에 대해 실행해야 함을 기억하세요: `worker-0`, `worker-1`, `worker-2`

## 검증

> 이 튜토리얼에서 생성된 compute instance들은 이 섹션을 할 수 있는 권한을 가지고 있지 않습니다. 다음의 명령어들은 compute instance들을 생성했을 때 사용했던 것과 동일한 머신에서 진행하시기 바랍니다.

> 역자 주) 로컬 컴퓨터에서 진행하시면 됩니다.

등록된 쿠버네티스 노드 리스트를 확인하세요:

```bash
gcloud compute ssh controller-0 \
  --command "kubectl get nodes --kubeconfig admin.kubeconfig"
```

> 결과

```
NAME       STATUS   ROLES    AGE   VERSION
worker-0   Ready    <none>   15s   v1.15.3
worker-1   Ready    <none>   15s   v1.15.3
worker-2   Ready    <none>   15s   v1.15.3
```

다음: [Configuring kubectl for Remote Access](10-configuring-kubectl.md)
