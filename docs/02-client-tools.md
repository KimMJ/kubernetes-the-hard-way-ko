# Client Tools 설치

In this lab you will install the command line utilities required to complete this tutorial: [cfssl](https://github.com/cloudflare/cfssl), [cfssljson](https://github.com/cloudflare/cfssl), and [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl).

이 랩에서는 이 튜토리얼을 성공적으로 마치기 위해 필요한 명령어 도구를 설치할 것입니다: [cfssl](https://github.com/cloudflare/cfssl), [cfssljson](https://github.com/cloudflare/cfssl), [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl).


## CFSSL 설치

명령어 도구 `cfssl`과 `cfssljson`는 [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure)를 설정하고 TLS certificates를 생성하는 데 사용됩니다.

`cfssl`과 `cfssljson`을 다운로드하고 설치하세요:

### OS X

```bash
curl -o cfssl https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/darwin/cfssl
curl -o cfssljson https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/darwin/cfssljson
```

```bash
chmod +x cfssl cfssljson
```

```bash
sudo mv cfssl cfssljson /usr/local/bin/
```

Some OS X users may experience problems using the pre-built binaries in which case [Homebrew](https://brew.sh) might be a better option:

몇몇 OS X 유저들은 미리 빌드된 바이너리를 사용하는 데 문제를 겪고 있습니다. 이 경우에는 [Homebrew](https://brew.sh)가 더 좋은 선택지가 될 것입니다:

```bash
brew install cfssl
```

### Linux

```bash
wget -q --show-progress --https-only --timestamping \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/linux/cfssl \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/linux/cfssljson
```

```bash
chmod +x cfssl cfssljson
```

```bash
sudo mv cfssl cfssljson /usr/local/bin/
```

### 검증

`cfssl`과 `cfssljson`가 1.3.4이상의 버전이  설치되었는지 확인합니다:

```bash
cfssl version
```

> 결과

```
Version: 1.3.4
Revision: dev
Runtime: go1.13
```

```bash
cfssljson --version
```
```
Version: 1.3.4
Revision: dev
Runtime: go1.13
```

## kubectl 설치

The `kubectl` command line utility is used to interact with the Kubernetes API Server. Download and install `kubectl` from the official release binaries:

명령어 도구 `kubectl`은 쿠버네티스의 API Server와 통신하는 데 사용되는 도구입니다. `kubectl`을 공식 릴리즈 바이너리를 통해 다운받고 설치합니다:

> 역자 주) 본 과정은 kubectl 1.15.3 버전을 다운로드 받는 과정입니다. 이미 다른 버전으로 깔려있을 경우 너무 많이 차이가 나지 않는 이상 크게 문제되지는 않을 것이라 생각됩니다.

### OS X

```bash
curl -o kubectl https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/darwin/amd64/kubectl
```

```bash
chmod +x kubectl
```

```bash
sudo mv kubectl /usr/local/bin/
```

### Linux

```bash
wget https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kubectl
```

```bash
chmod +x kubectl
```

```bash
sudo mv kubectl /usr/local/bin/
```

### 검증

`kubectl`이 1.15.3 이상의 버전으로 설치되어 있는지 확인합니다:

```bash
kubectl version --client
```

> 결과

```
Client Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.3", GitCommit:"2d3c76f9091b6bec110a5e63777c332469e0cba2", GitTreeState:"clean", BuildDate:"2019-08-19T11:13:54Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}
```

다음: [Provisioning Compute Resources](03-compute-resources.md)
