# Compute Resources 설정

쿠버네티스는 쿠버네티스 control plane과 최종적으로 컨테이너가 동작하게 될 worker node들을 호스트들을 필요로 합니다. 이 랩에서는 단일 [compute zone](https://cloud.google.com/compute/docs/regions-zones/regions-zones)에서 동작하는 안전하고 고가용성이 가능한 쿠버네티스 클러스터를 동작시키는 데 필요한 compute resources를 설정하게 될 것입니다.

> [Prerequisites](01-prerequisites.md#set-a-default-compute-region-and-zone)에서 설명한 바와 같이 default compute zone과 region이 설정되어 있는지 확인하시길 바랍니다.

## Networking

쿠버네티스의 [네트워킹 모델(networking model)](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model)은 노드와 컨테이너가 서로 통신할 수 있는 평평한 네트워크라고 가정합니다. 이것이 원하지 않는  [네트워크 정책(network policies)](https://kubernetes.io/docs/concepts/services-networking/network-policies/)인 경우 컨테이너 그룹들이 서로간에, 그리고 외부 네트워크 엔드 포인트간의 통신을 하는 것에 제한을 둘 수 있습니다.

> 네트워크 정책을 설정하는 것은 이 튜토리얼의 범위 밖입니다.

### Virtual Private Cloud Network

이 섹션에서는 쿠버네티스 클러스터를 호스팅하기 위해  전용 [Virtual Private Cloud](https://cloud.google.com/compute/docs/networks-and-firewalls#networks) (VPC) 네트워크를 설정합니다.

`kubernetes-the-hard-way`라는 커스텀 VPC 네트워크를 생성합니다:

```bash
gcloud compute networks create kubernetes-the-hard-way --subnet-mode custom
```

> 역자 주) 위의 명령어에서 에러가 날 경우 화면에 뜨는 url로 접속하시기 바랍니다. 접속 후 Compute Engine API에 대한 결제 정보를 확인한 후 다시 입력하면 정상적으로 동작합니다.

[subnet](https://cloud.google.com/compute/docs/vpc/#vpc_networks_and_subnets)은 쿠버네티스 클러스터의 각 노드에 대해 사설 IP 주소를 할당하기에 충분한 크기의 IP 주소 범위를 가지도록 설정해야 합니다.

`kubernetes-the-hard-way` VPC 네트워크에서 `kubernetes` subnet을 생성합니다:

```bash
gcloud compute networks subnets create kubernetes \
  --network kubernetes-the-hard-way \
  --range 10.240.0.0/24
```

> `10.240.0.0/24` IP 주소 범위는 254개의 compute instance들을 호스팅 할 수 있습니다.

### Firewall Rules

방화벽 규칙을 생성하여 모든 프로토콜에 대해 내부 통신을 허용합니다.

```bash
gcloud compute firewall-rules create kubernetes-the-hard-way-allow-internal \
  --allow tcp,udp,icmp \
  --network kubernetes-the-hard-way \
  --source-ranges 10.240.0.0/24,10.200.0.0/16
```

방화벽 규칙을 생성하여 외부 SSH, ICMP, HTTPS를 허용합니다.

```bash
gcloud compute firewall-rules create kubernetes-the-hard-way-allow-external \
  --allow tcp:22,tcp:6443,icmp \
  --network kubernetes-the-hard-way \
  --source-ranges 0.0.0.0/0
```

> [외부 load balancer](https://cloud.google.com/compute/docs/load-balancing/network/)는 쿠버네티스 API Servers를 원격 클라이언트에 대해 노출시키는 데 사용될 수 있습니다.

`kubernetes-the-hard-way` VPC 네트워크에서의 방화벽 규칙의 리스트를 볼 수 있습니다:

```bash
gcloud compute firewall-rules list --filter="network:kubernetes-the-hard-way"
```

> 결과

```
NAME                                    NETWORK                  DIRECTION  PRIORITY  ALLOW                 DENY
kubernetes-the-hard-way-allow-external  kubernetes-the-hard-way  INGRESS    1000      tcp:22,tcp:6443,icmp
kubernetes-the-hard-way-allow-internal  kubernetes-the-hard-way  INGRESS    1000      tcp,udp,icmp
```

### Kubernetes Public IP Address

쿠버네티스 API Servers의 앞단에 있는 외부 load balancer에 연결할 정적 IP 주소를 할당합니다:

```bash
gcloud compute addresses create kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region)
```

Verify the `kubernetes-the-hard-way` static IP address was created in your default compute region:

`kubernetes-the-hard-way`의 정적 IP wnthrk default compute region 안에 생성되었는지 확인합니다:

```bash
gcloud compute addresses list --filter="name=('kubernetes-the-hard-way')"
```

> 결과

```
NAME                     REGION    ADDRESS        STATUS
kubernetes-the-hard-way  us-west1  XX.XXX.XXX.XX  RESERVED
```

## Compute Instances

이 랩에서 compute instance들은 [containerd container runtime](https://github.com/containerd/containerd)을 잘 지원하는 [Ubuntu Server](https://www.ubuntu.com/server) 18.04를 사용하도록 설정합니다. 각 compute instance는 쿠버네티스 부트스트랩 과정을 간단하게 하기 위해 고정된 사설 IP 주소를 사용합니다.

> 역자 주, [wikipedia]([https://ko.wikipedia.org/wiki/%EB%B6%80%ED%8A%B8%EC%8A%A4%ED%8A%B8%EB%9E%A9](https://ko.wikipedia.org/wiki/부트스트랩)) 참조) 부트스트랩 : 더 복잡한 도구를 만들 수 있도록 도와 주는 단순 도구를 만들거나 적재함으로써 복잡한 [소프트웨어](https://ko.wikipedia.org/wiki/소프트웨어) 도구를 만들거나 컴퓨터를 시작하는 것을 말한다. 줄여서 [시동](https://ko.wikipedia.org/wiki/시동)이라고도 할 수 있으며, 이는 컴퓨터를 시작하는 과정을 서술해 준다.

### Kubernetes Controllers

쿠네티스 control plane을 호스팅 할 세개의 compute instance들을 생성합니다:

```bash
for i in 0 1 2; do
  gcloud compute instances create controller-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --private-network-ip 10.240.0.1${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,controller
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

각 worker instance는 쿠버네티스 클러스터 CIDR의 범위로부터 파드 서브넷 할당을 필요로 합니다. 파드 서브넷 할당은 나중에 컨테이너 네트워킹을 설정하는 데 사용될 것입니다. `pod-cidr` instance 메타데이터는 파드 서브넷 할당을 런타임에 compute instance들로 노출시키는 데 사용됩니다.

> 쿠버네티스 클러스터 CIDR의 범위는 컨트롤러 매니저의 `--cluster-cidr` 플래그에 의해 정의됩니다. 이 튜토리얼에서 CIDR의 범위는 254개의 서브넷을 지원하는 `10.200.0.0/16`으로 설정됩니다.

> 역자 주) 10.200.0.0/16은 10.200.0.1 ~ 10.200.255.254 범위의 IP 주소를 가져 65534개의 서브넷을 가진다고 생각되지만 왜 글쓴이가 254개의 서브넷을 지원한다고 했는지 모르겠습니다.

쿠버네티스 worker node들을 호스팅할 세개의 compute instance들을 생성합니다:

```bash
for i in 0 1 2; do
  gcloud compute instances create worker-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --metadata pod-cidr=10.200.${i}.0/24 \
    --private-network-ip 10.240.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,worker
done
```

>  역자 주) 할당량 제한으로 VM이 정상적으로 생성되지 않을 수 있습니다. `GCP > IAM 및 관리자 > 할당량`으로 이동하여 us-west1에 대해 In-use IP addresses 할당량을 6으로 높이시면 됩니다.

### 검증

default compute zone에서 compute instance들의 리스트를 확인합니다:

```bash
gcloud compute instances list
```

> 결과

```
NAME          ZONE        MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
controller-0  us-west1-c  n1-standard-1               10.240.0.10  XX.XXX.XXX.XXX  RUNNING
controller-1  us-west1-c  n1-standard-1               10.240.0.11  XX.XXX.X.XX     RUNNING
controller-2  us-west1-c  n1-standard-1               10.240.0.12  XX.XXX.XXX.XX   RUNNING
worker-0      us-west1-c  n1-standard-1               10.240.0.20  XXX.XXX.XXX.XX  RUNNING
worker-1      us-west1-c  n1-standard-1               10.240.0.21  XX.XXX.XX.XXX   RUNNING
worker-2      us-west1-c  n1-standard-1               10.240.0.22  XXX.XXX.XX.XX   RUNNING
```

## SSH 접속 설정

SSH는 controller와 worker instance들을 설정하는 데 사용될 것입니다.[connecting to instances](https://cloud.google.com/compute/docs/instances/connecting-to-instance) 문서에 나와있듯이 compute instance들에  처음 접속할 때 SSH key들이 생성되고 프로젝트 내부 또는 instance 메타데이터에 저장됩니다.

`controller-0` compute instance로의 SSH 접속을 테스트를 합니다:

```bash
gcloud compute ssh controller-0
```

처음으로 compute instance에 접속하면 SSH key들이 생성됩니다. 프롬프트에서 passphrase를 입력하여 계속 진행합니다.

```
WARNING: The public SSH key file for gcloud does not exist.
WARNING: The private SSH key file for gcloud does not exist.
WARNING: You do not have an SSH key for gcloud.
WARNING: SSH keygen will be executed to generate a key.
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
```

여기서 생성된 SSH key들은 당신의 프로젝트 안에 업로드 되고 저장됩니다.

```
Your identification has been saved in /home/$USER/.ssh/google_compute_engine.
Your public key has been saved in /home/$USER/.ssh/google_compute_engine.pub.
The key fingerprint is:
SHA256:nz1i8jHmgQuGt+WscqP5SeIaSy5wyIJeL71MuV+QruE $USER@$HOSTNAME
The key's randomart image is:
+---[RSA 2048]----+
|                 |
|                 |
|                 |
|        .        |
|o.     oS        |
|=... .o .o o     |
|+.+ =+=.+.X o    |
|.+ ==O*B.B = .   |
| .+.=EB++ o      |
+----[SHA256]-----+
Updating project ssh metadata...-Updated [https://www.googleapis.com/compute/v1/projects/$PROJECT_ID].
Updating project ssh metadata...done.
Waiting for SSH key to propagate.

```

SSH key들이 업데이트 되고 나면 `controller-0` instance로 로그인 될 것입니다.

```
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-1042-gcp x86_64)
...

Last login: Sun Sept 14 14:34:27 2019 from XX.XXX.XXX.XX
```

`controller-0` compute instance에서 나가기 위해 프롬프트에서 `exit`를 입력합니다.

```
$USER@controller-0:~$ exit
```
> 결과

```
logout
Connection to XX.XXX.XXX.XXX closed
```

> 역자 주) 다음으로 진행하기 전에 controller-0,1,2 및 worker-0,1,2에 대해 미리 ssh 접속을 한번씩 하시기 바랍니다.

다음: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)

