# AWS EKS 배포

## kubectl 설치

최신 릴리즈 다운로드

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

kubectl 설치

```bash
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

## aws cli 설치

```bash
sudo apt install unzip

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Access Key, Private Key 등록
aws configure
```

## Helm 설치

최신 릴리즈 다운로드 후 설치

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

## eksctl 설치

최신 릴리즈 다운로드

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
```

압축 해제된 이진 파일을  `/usr/local/bin`으로 옮김

```bash
sudo mv /tmp/eksctl /usr/local/bin
```

## AWS EKS

```bash
aws configure
eksctl create cluster --name myeks --nodes=3 --region=ap-northeast-2
```

> 안되는 것들
> 
> Load Balancer Service = class lb -> nlb
> 
> Ingress: X
> 
> kubectl top: X -> HPA X

---

## YAML 파일을 이용한 EKS 배포

```bash
mkdir aws-eks
cd aws-eks
```

`myeks.yaml`

```bash
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: myeks-custom
  region: ap-northeast-2
  version: "1.24"

# AZ
availabilityZones: ["ap-northeast-2a", "ap-northeast-2c"]

# IAM OIDC & Service Account
iam:
  withOIDC: true
  serviceAccounts:
    - metadata:
        name: aws-load-balancer-controller
        namespace: kube-system
      wellKnownPolicies:
        awsLoadBalancerController: true
    - metadata:
        name: ebs-csi-controller-sa
        namespace: kube-system
      wellKnownPolicies:
        ebsCSIController: true
    - metadata:
        name: cluster-autoscaler
        namespace: kube-system
      wellKnownPolicies:
        autoScaler: true

# Managed Node Groups
managedNodeGroups:
  # On-Demand Instance
  - name: myeks-ng1
    instanceType: t3.medium
    spot: true
    minSize: 3
    desiredCapacity: 3
    maxSize: 5
    privateNetworking: true
    ssh:
      allow: true
      publicKeyPath: ./keypair/myeks.pub
    availabilityZones: ["ap-northeast-2a", "ap-northeast-2c"]
    iam:
      withAddonPolicies:
        autoScaler: true
        albIngress: true
        cloudWatch: true
        ebs: true

# Fargate Profiles
fargateProfiles:
  - name: fg-1
    selectors:
    - namespace: dev
      labels:
        env: fargate

# CloudWatch Logging
cloudWatch:
  clusterLogging:
    enableTypes: ["*"]
```

```bash
mkdir keypair
ssh-keygen -f keypair/myeks
```

```bash
eksctl create cluster -f myeks.yaml
```

## NLB for LoadBalancer Service

> https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/network-load-balancing.htmlhttps://docs.aws.amazon.com/ko_kr/eks/latest/userguide/alb-ingress.htmlhttps://docs.aws.amazon.com/ko_kr/eks/latest/userguide/aws-load-balancer-controller.html
> 

### AWS Load Balancer Controller 설치

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=myeks-custom --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller --set image.repository=602401143452.dkr.ecr.ap-northeast-2.amazonaws.com/amazon/aws-load-balancer-controller
```

## 샘플 코드

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myweb-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: myweb
          image: ghcr.io/c1t1d0s7/go-myweb
          ports:
            - containerPort: 8080
```

```bash
apiVersion: v1
kind: Service
metadata:
  name: myweb-svc-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "instance"
	service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
```

- [service.beta.kubernetes.io/aws-load-balancer-nlb-target-type](http://service.beta.kubernetes.io/aws-load-balancer-nlb-target-type)
    - instance: EC2 타겟
    - ip: Pod 타겟(Fargate)
- [service.beta.kubernetes.io/aws-load-balancer-scheme](http://service.beta.kubernetes.io/aws-load-balancer-scheme)
    - internal: 내부
    - internet-facing: 외부

## Ingress for ALB

> https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/alb-ingress.html
> 

```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myweb-ing
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/target-type: instance
    alb.ingress.kubernetes.io/scheme: internet-facing
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myweb-svc-lb
                port:
                  number: 80
```

- [alb.ingress.kubernetes.io/target-type](http://alb.ingress.kubernetes.io/target-type)
    - instance: EC2 타겟
    - ip: Pod 타겟(Fargate)
- [alb.ingress.kubernetes.io/scheme](http://alb.ingress.kubernetes.io/scheme)
    - internal: 내부
    - internet-facing: 외부

## EBS for CSI

- EBS 스냅샷
- EBS 크기 변경

> https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/managing-ebs-csi.html
> 

```bash
eksctl get iamserviceaccount --cluster myeks-custom

NAMESPACE       NAME                            ROLE ARN
kube-system     aws-load-balancer-controller    arn:aws:iam::065144736597:role/eksctl-myeks-custom-addon-iamserviceaccount-Role1-11N0OKMVG2DYY
kube-system     aws-node                        arn:aws:iam::065144736597:role/eksctl-myeks-custom-addon-iamserviceaccount-Role1-CLMK7A6K5NL3
kube-system     cluster-autoscaler              arn:aws:iam::065144736597:role/eksctl-myeks-custom-addon-iamserviceaccount-Role1-1S02W28MZOSL4
kube-system     ebs-csi-controller-sa           arn:aws:iam::065144736597:role/eksctl-myeks-custom-addon-iamserviceaccount-Role1-15HLE8HBOD9CN
```

```bash
eksctl create addon --name aws-ebs-csi-driver --cluster myeks-custom --service-account-role-arn  arn:aws:iam::065144736597:role/eksctl-myeks-custom-addon-iamserviceaccount-Role1-15HLE8HBOD9CN --force
```

## Metrics Server

> https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/metrics-server.html
> 

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

## Cluster Autoscaler

### 수동 스케일링

```bash
eksctl scale nodegroup --name myeks-ng1 --cluster myeks-custom --nodes 2
```

### 자동 스케일링

> https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/autoscaling.html
> 

```bash
curl -o cluster-autoscaler-autodiscover.yaml https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
```

`cluster-autoscaler-autodiscover.yaml`

```
163: - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/myeks-custom
```

```bash
kubectl apply -f cluster-autoscaler-autodiscover.yaml
```

```bash
kubectl patch deployment cluster-autoscaler -n kube-system -p '{"spec":{"template":{"metadata":{"annotations":{"cluster-autoscaler.kubernetes.io/safe-to-evict": "false"}}}}}'
```

```bash
kubectl -n kube-system edit deployment.apps/cluster-autoscaler
```

```
      - command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/myeks-custom
        - --balance-similar-node-groups
        - --skip-nodes-with-system-pods=false
        image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.22.6
```

수정

- -node-group-auto-discovery=asg:tag=[k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/myeks-custom](http://k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/myeks-custom)
- -balance-similar-node-groups
- -skip-nodes-with-system-pods=false
- image: [k8s.gcr.io/autoscaling/cluster-autoscaler:v1.22.2](http://k8s.gcr.io/autoscaling/cluster-autoscaler:v1.22.2)

```
kubectl set image deployment cluster-autoscaler -n kube-system cluster-autoscaler=k8s.gcr.io/autoscaling/cluster-autoscaler:v1.22.2
```

### 샘플 코드

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myweb-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: myweb
          image: ghcr.io/c1t1d0s7/go-myweb:alpine
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 200m
              memory: 200M
            limits:
              cpu: 200m
              memory: 200M
```

## CloudWatch Container Insight

> https://docs.aws.amazon.com/ko_kr/AmazonCloudWatch/latest/monitoring/Container-Insights-setup-EKS-quickstart.html
> 

> https://github.com/git-for-windows/git/releases/download/v2.36.1.windows.1/Git-2.36.1-64-bit.exe
> 

```
ClusterName=myeks-custom
RegionName=ap-northeast-2
FluentBitHttpPort='2020'
FluentBitReadFromHead='Off'
[[ ${FluentBitReadFromHead} = 'On' ]] && FluentBitReadFromTail='Off'|| FluentBitReadFromTail='On'
[[ -z ${FluentBitHttpPort} ]] && FluentBitHttpServer='Off' || FluentBitHttpServer='On'
curl <https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart.yaml> | sed 's/{{cluster_name}}/'${ClusterName}'/;s/{{region_name}}/'${RegionName}'/;s/{{http_server_toggle}}/"'${FluentBitHttpServer}'"/;s/{{http_server_port}}/"'${FluentBitHttpPort}'"/;s/{{read_from_head}}/"'${FluentBitReadFromHead}'"/;s/{{read_from_tail}}/"'${FluentBitReadFromTail}'"/' | kubectl apply -f -
```

## Fargate

EC2 인스턴스 사용 , 파드를 실행

> https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/fargate.html
> 

```bash
kubectl create ns dev
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myfg
  namespace: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myfg
  template:
    metadata:
      labels:
        app: myfg
        env: fargate
    spec:
      containers:
      - name: myfg
        image: ghcr.io/c1t1d0s7/go-myweb
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 8080
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysvc
  namespace: dev
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
spec:
  selector:
    app: myfg
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

## VPA

> https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/vertical-pod-autoscaler.html
> 

사전 요구 사항

- openssl 1.1.1 이상
- metrics-server

```bash
git clone <https://github.com/kubernetes/autoscaler.git>
```

```bash
cd autoscaler/vertical-pod-autoscaler/
```

```bash
/hack/vpa-up.sh
```

```bash
kubectl get pods -n kube-system
```

### VPA 예제

```bash
kubectl apply -f examples/hamster.yaml
```

## 클러스터 삭제

```bash
eksctl delete cluster -f .\\myeks.yaml --force --disable-nodegroup-eviction
```
