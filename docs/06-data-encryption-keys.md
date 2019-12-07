# Data Encryption Config와 Key 생성하기

쿠버네티스는 클러스터의 상태, 어플리케이션의 설정, 시크릿을 포함한 다양한 데이터를 저장합니다. 쿠버네티스는 유휴 상태의 클러스터 데이터를 [암호화](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data)할 수 있는 기능을 지원합니다.

이 랩에서는 쿠버네티스 시크릿을 암호화하는 데 적합한 encryption key와 [encryption config](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#understanding-the-encryption-at-rest-configuration)를 생성합니다. 

## The Encryption Key

encryption key를 생성합니다:

```bash
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

## The Encryption Config File

`encryption-config.yaml` encryption config 파일을 생성합니다:

```bash
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

`encryption-config.yaml` encryption config file을 각 controller instance에 복사합니다:

```bash
for instance in controller-0 controller-1 controller-2; do
  gcloud compute scp encryption-config.yaml ${instance}:~/
done
```

다음: [Bootstrapping the etcd Cluster](07-bootstrapping-etcd.md)
