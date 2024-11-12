# EKS Blue/Green 업그레이드

기존 클러스터의 모든 리소스를 새로운 클러스터로 복제하고, 트래픽을 점진적으로 전환하는 작업을 통해 무중단 업그레이드를 진행한다.

## 1. Prerequisite

```bash
# Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# EFS CSI Driver
eksctl get iamserviceaccount --cluster myeks-blue
eksctl create addon --name aws-efs-csi-driver \ 
      --cluster myeks-blue \ 
      --service-account-role-arn arn:aws:iam::<ACCOUNT_ID>:role/eksctl-myeks-blue-addon-iamserviceaccount-kub-Role1-z3uj8CQJjk4N

git clone https://github.com/kubernetes-sigs/aws-efs-csi-driver.git
cd aws-efs-csi-driver/examples/kubernetes/dynamic_provisioning/specs
kubectl apply -f storageclass.yaml
```

## 2. EKS 클러스터 생성

EKS 클러스터의 생성은 다음 문서로 대체한다.

[AWS EKS 배포](./AWS_EKS_%EB%B0%B0%ED%8F%AC.md)

[yaml 참고](./aws-eks/myeks.yaml)

## 3. 네트워크 및 IAM 설정 복제

기존 클러스터에서 사용하던 네트워크(VPC, 서브넷, 보안 그룹) 및 IAM 역할을 새 클러스터에도 동일하게 설정한다.

특히 Service Account와 연동되는 IAM 역할을 새 클러스터에도 구성하여 권한이 일치하도록 설정한다.

## 4. ArgoCD를 사용한 애플리케이션 동기화

GitOps 방식으로, 기존 Blue 클러스터의 리소스 설정을 Git 레포지토리에서 가져와 Green 클러스터에 배포할 수 있다. 이를 위해 ArgoCD에서 Green 클러스터를 새로운 클러스터로 등록하고, 필요한 애플리케이션을 동기화한다.

### 4.1. Blue 클러스터 리소스 배포

(배포 리소스)[https://github.com/ddung1203/youtube-jenkins/tree/dev/k8s-manifest/helm-charts/youtube]


```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 4.2. Green 클러스터 설정

ArgoCD 서버 주소 설정

```bash
argocd login <BLUE's ARGOCD SVC>
```

ArgoCD 클러스터 추가

```bash
argocd cluster add <GREEN's CLUSTER_NAME>
```

애플리케이션의 대상 클러스터 변경

```bash
argocd app set youtube --dest-server https://<GREEN's CLUSTER_URL> --dest-namespace youtube
```

애플리케이션 배포 및 동기화

```bash
argocd app sync youtube
```

