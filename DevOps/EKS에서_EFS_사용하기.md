# EKS에서 EFS 사용하기

AWS EKS에서 여러 Pod들이 하나의 스토리지를 사용하도록 할 수 있습니다.
가장 접근하기 쉬운 EBS 볼륨을 사용할 경우, EBS의 특성상 다른 AZ에서는 접근할 수 없다는 단점이 있어 여러 가용 영역에 Worker Node가 배포되어 있을 경우, EBS가 위치하지 않은 AZ에 있는 Worker Node에는 해당 EBS를 스토리지로 사용하는 Pod들이 배포되지 않습니다.
이를 해결하기 위해 다른 AZ에서도 사용이 가능한 EFS를 활용할 수 있습니다.
AWS EKS에서 EFS를 사용하기 위해서는 몇가지 설정이 필요합니다.
EKS에 EFS를 사용할 수 있는 드라이버가 설치되어있어야 하며,Worker Node들이 EFS를 사용할 수 있는 AmazonElasticFileSystemFullAccess 권한을 소유하고 있어야 합니다.

1. EKS CSI 드라이버 설치

```bash
# helm 설치
$ curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh
$ chmod 700 get_helm.sh
$ ./get_helm.sh

# 아래 에러 발생시 vi get_helm.sh을 입력한 후
# 편집기에서 HELM_INSTALL_DIR을 $PATH와 일치하도록 변경
helm not found. Is /usr/local/bin on your $PATH?
Failed to install helm
        For support, go to https://github.com/helm/helm.

# helm에 aws-efs-csi-driver repository 추가
$ helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/

# repository 업데이트
$ helm repo update

# 드라이버 설치
# https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/add-ons-images.html
$ helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
    --namespace kube-system \
    --set image.repository={위의 링크에서 리전별 값 확인}/eks/aws-efs-csi-driver \
    --set controller.serviceAccount.create=false \
    --set controller.serviceAccount.name=efs-csi-controller-sa
```

1. EFS 생성

```bash
# 1. EKS 클러스터의 vpc_id 값 조회
$ vpc_id=$(aws eks describe-cluster \
    --name {EKS 클러스터 명} \
    --query "cluster.resourcesVpcConfig.vpcId" \
    --output text)
    
# 2. EKS 클러스터가 속한 vpc의 network cidr 조회
$ cidr_range=$(aws ec2 describe-vpcs \
    --vpc-ids $vpc_id \
    --query "Vpcs[].CidrBlock" \
    --output text)

# 3. EFS의 보안그룹 생성
$ security_group_id=$(aws ec2 create-security-group \
    --group-name {보안그룹 명} \
    --description "{보안그룹 설명}" \
    --vpc-id $vpc_id \
    --output text)

# 4. 보안그룹에 인바운드 정책 추가
$ aws ec2 authorize-security-group-ingress \
    --group-id $security_group_id \
    --protocol tcp \
    --port 2049 \
    --cidr $cidr_range
    
# 5. EFS 생성
$ file_system_id=$(aws efs create-file-system \
    --region {region-code} \
    --performance-mode generalPurpose \
    --tag Key=Name,Value={EFS 이름} \
    --query 'FileSystemId' \
    --output text)
```