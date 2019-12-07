# Kubernetes The Hard Way

이 튜토리얼에서는 쿠버네티스를 약간은 어려운 방법으로 설정하는 것을 배웁니다. 이 가이드는 쿠버네티스 클러스터를 생성하는 완전히 자동화된 커맨드를 찾으려는 사람들에게는 적합하지 않습니다. 만약 그런 커맨드를 찾고자 한다면 [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine) 또는 [Getting Started Guides](https://kubernetes.io/docs/setup)를 확인하시기 바랍니다.

Kubernetes The Hard Way는 학습에 특화되어 있으며 이는 쿠버네티스 클러스터를 부트스트랩 하는데 필요한 각 task를 이해하는 데 긴 시간이 필요함을 의미합니다.

> 이 튜토리얼에서 나온 결과물은 실제 프로덕션을 고려할 때 완벽하다고 볼 수 없으며 커뮤니티로부터 지원을 받는데 제한이 있을 수 있습니다. 하지만 배움을 멈추지는 마십시오!

## Copyright

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a><br />This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License</a>.


## Target Audience

이 튜토리얼의 타겟은 어떻게 모든 것들이 작동하는지 이해하고 싶어하는, 프로덕션 쿠버네티스 클러스터를 지원해야 하는 누군가를 위한 것입니다.

## Cluster Details

Kubernetes The Hard Way는 end-to-end 컴포넌트 사이의 암호화와 RBAC 인증을 하는, 고가용성을 지원하는 쿠버네티스 클러스터를 부트스트래핑하도록 가이드할 것입니다.

* [kubernetes](https://github.com/kubernetes/kubernetes) 1.15.3
* [containerd](https://github.com/containerd/containerd) 1.2.9
* [coredns](https://github.com/coredns/coredns) v1.6.3
* [cni](https://github.com/containernetworking/cni) v0.7.1
* [etcd](https://github.com/coreos/etcd) v3.4.0

## Labs

이 튜토리얼에서는 [Google Cloud Platform](https://cloud.google.com)에 접근 권한이 있다고 가정합니다. GCP가 기본 infrastructure로 사용되고 있지만 이 튜토리얼에서 배운 점들은 다른 플랫폼에 대해서도 적용할 수 있습니다. 

* [Prerequisites](docs/01-prerequisites.md)
* [Installing the Client Tools](docs/02-client-tools.md)
* [Provisioning Compute Resources](docs/03-compute-resources.md)
* [Provisioning the CA and Generating TLS Certificates](docs/04-certificate-authority.md)
* [Generating Kubernetes Configuration Files for Authentication](docs/05-kubernetes-configuration-files.md)
* [Generating the Data Encryption Config and Key](docs/06-data-encryption-keys.md)
* [Bootstrapping the etcd Cluster](docs/07-bootstrapping-etcd.md)
* [Bootstrapping the Kubernetes Control Plane](docs/08-bootstrapping-kubernetes-controllers.md)
* [Bootstrapping the Kubernetes Worker Nodes](docs/09-bootstrapping-kubernetes-workers.md)
* [Configuring kubectl for Remote Access](docs/10-configuring-kubectl.md)
* [Provisioning Pod Network Routes](docs/11-pod-network-routes.md)
* [Deploying the DNS Cluster Add-on](docs/12-dns-addon.md)
* [Smoke Test](docs/13-smoke-test.md)
* [Cleaning Up](docs/14-cleanup.md)
