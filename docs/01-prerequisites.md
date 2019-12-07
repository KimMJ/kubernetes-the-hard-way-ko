# Prerequisites

## Google Cloud Platform

This tutorial leverages the [Google Cloud Platform](https://cloud.google.com/) to streamline provisioning of the compute infrastructure required to bootstrap a Kubernetes cluster from the ground up. [Sign up](https://cloud.google.com/free/) for $300 in free credits.

이 튜토리얼은 [Google Cloud Platform](https://cloud.google.com/)을 활용하여 처음부터 쿠버네티스 클러스터를 부팅하는 데 필요한 컴퓨터 인프라의 설정을 간소화합니다. [가입](https://cloud.google.com/free/)하여 $300의 무료 크레딧을 받으세요.

튜토리얼을 진행하는 데 사용되는 [예상 가격](https://cloud.google.com/products/calculator/#id=55663256-c384-449c-9306-e39893e23afb) 은 시간당 \$0.23입니다. (하루에 \$5.46 달러)

> 이 튜토리얼을 진행하는 데 필요한 리소스는 무료 기준을 초과합니다. 중간에 중단할 경우 compute resource를 중단시켜 가격을 아끼시기 바랍니다.

## Google Cloud Platform SDK

### Google Cloud SDK 설치

Google Cloud SDK [문서](https://cloud.google.com/sdk/)를 참조하여 `gcloud` 명령어 도구를 설치하고 설정하시기 바랍니다.

Google Cloud SDK의 버전이 262.0.0 또는 더 높은지 확인하세요. :

```bash
gcloud version
```

### Default Compute Region과 Zone 설정

이 튜토리얼은 default compute region과 zone이 이미 설정되어있다고 가정합니다.

만약 `gcloud` 명령어 도구를 처음 사용한다면 `init`은 이를 설정하는 가장 쉬운 방법일 것입니다:

```bash
gcloud init
```

그 후 `gcloud`가 당신의 Google user credentials를 통해 Cloud Platform에 접속할 권한을 가지고 있는지 확인합니다:

```bash
gcloud auth login
```

다음으로는 default compute region과 compute zone을 설정합니다:

```bash
gcloud config set compute/region us-west1
```

default compute zone을 설정합니다:

```bash
gcloud config set compute/zone us-west1-c
```

> `gcloud compute zones list` 명령어를 사용하여 추가적인 region이나 zone들을 확인할 수 있습니다.

> 역자 주) 할당량 제한으로 중간에 오류가 발생할 것입니다. `GCP > IAM 및 관리자 > 할당량`으로 이동하여 us-west1에 대해 In-use IP addresses 할당량을 6으로 높이시면 됩니다.

## tmux를 통해 명령어를 동시에 실행하기

[tmux](https://github.com/tmux/tmux/wiki)를 통해 여러 compute instance들에 대해서 동시에 명령어를 입력할 수 있습니다. 이 튜토리얼에서는 같은 명령어를 여러 compute instance들에 대해 실행시켜야 하고 이런 경우에 tmux를 사용하여 창을 여러개의 pane(판)으로 나누어 synchronize-panes를 활성화 시키면 설정 프로세스에 속도를 낼 수 있습니다.

> tmux를 사용하는 것은 개인의 자유이며 이 튜토리얼을 마치는 데 필수적인 것은 아닙니다.

![tmux screenshot](images/tmux-screenshot.png)

> 판에 동시 입력을 활성화 하려면 `ctrl+b` 를 누른 뒤 `shift+:`를 누르십시오. 그 다음 `set synchronized-panes on`을 프롬프트에 입력하세요. 동시 입력을 비활성화 하려면 `set synchronize-panes off`를 입력하면 됩니다.

> 역자 주) tmux 간단 사용법
>
> 1. 가로로 분할
>
>    ```tmux
>    ctrl + b, "
>    ```
>
> 2. 세로로 분할
>
>    ```tmux
>    ctrl + b, %
>    ```
>
> 3. 가로로 균등 정렬
>
>    ```tmux
>    ctrl + b, alt + 2
>    ```
>
> 4. 세로로 균등 정렬
>
>    ```tmux
>    ctrl + b, alt + 1
>    ```
>
> 5. 동시에 입력 활성화
>
>    ```tmux
>    ctrl + b, :set synchronize-panes on
>    ```
>
> 6. 동시에 입력 비활성화
>
>    ```tmux
>    ctrl + b, :set synchronize-panes off
>    ```

다음: [Client Tools 설치](02-client-tools.md)