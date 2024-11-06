# AWS Code Series

목차

* 1. [AWS Code Series를 이용한 자동화 구성](#1-aws-code-series를-이용한-자동화-구성)
* 2. [Requirements](#2-requirements)
* 3. [Deploy](#3-deploy)
  * 3.1. [ECS Deploy](#31-ecs-deploy)
    * 3.1.1. [ECS Task 접속 방법](#311-ecs-task-접속-방법)
  * 3.2. [Code Build](#32-code-build)
  * 3.3. [Code Pipeline](#33-code-pipeline)
    * 3.3.1. [Rolling Update](#331-rolling-update)
    * 3.3.2. [Blue/Green Update](#332-bluegreen-update)

---

## 1. AWS Code Series를 이용한 자동화 구성

사용된 Web Server : [YouTube Reloaded](https://github.com/ddung1203/youtube-reloaded)

컨테이너 실행 방법

```bash
docker network create youtube

docker run -d --name mongo --network youtube \ 
        -e MONGO_INITDB_ROOT_USERNAME=mongo  \
        -e MONGO_INITDB_ROOT_PASSWORD=mongo  \
        -p 27017:27017                       \
        -v /home/ubuntu/mongo:/data/db       \
        mongo

docker run -d --name youtube --network youtube   \
    -e MONGO_USERNAME=mongo                      \
    -e MONGO_PASSWORD=mongo                      \
    -e MONGO_URL=mongo                           \
    -e COOKIE_SECRET=youtube                     \
    -p 80:4000                                   \
    ddung1203/youtube:latest
```

## 2. Requirements

`ECS`, `ECR`, `EFS`, `Mongo`

## 3. Deploy

Code Build, Code Deploy, Code Pipeline을 테스트하기 위해 다음을 배포한다.

- `ECS`
- `ECR`
- `EFS`: [EFS 참고](../images/efs.png)
- `Mongo`

### 3.1. ECS Deploy

```json
{
  "taskDefinitionArn": "arn:aws:ecs:ap-northeast-2:<ACCOUNT_ID>:task-definition/VidTask:7",
  "containerDefinitions": [
    {
      "name": "youtube",
      "image": "<ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com/youtube:latest",
      "cpu": 0,
      "portMappings": [
        {
          "name": "youtube-4000-tcp",
          "containerPort": 4000,
          "hostPort": 4000,
          "protocol": "tcp",
          "appProtocol": "http"
        }
      ],
      "essential": true,
      "environment": [
        {
          "name": "MONGO_USERNAME",
          "value": "mongo"
        },
        {
          "name": "MONGO_URL",
          "value": "10.0.2.90"
        },
        {
          "name": "COOKIE_SECRET",
          "value": "youtube"
        },
        {
          "name": "MONGO_PASSWORD",
          "value": "mongo"
        }
      ],
      "environmentFiles": [],
      "mountPoints": [
        {
          "sourceVolume": "YouTube",
          "containerPath": "/app/uploads",
          "readOnly": false
        }
      ],
      "volumesFrom": [],
      "ulimits": [],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/VidTask",
          "mode": "non-blocking",
          "awslogs-create-group": "true",
          "max-buffer-size": "25m",
          "awslogs-region": "ap-northeast-2",
          "awslogs-stream-prefix": "ecs"
        },
        "secretOptions": []
      },
      "systemControls": []
    }
  ],
  "family": "VidTask",
  "taskRoleArn": "arn:aws:iam::<ACCOUNT_ID>:role/ecsTaskExecutionRole",
  "executionRoleArn": "arn:aws:iam::<ACCOUNT_ID>:role/ecsTaskExecutionRole",
  "networkMode": "awsvpc",
  "revision": 7,
  "volumes": [
    {
      "name": "YouTube",
      "efsVolumeConfiguration": {
        "fileSystemId": "fs-0a29dd8823cab60b5",
        "rootDirectory": "/",
        "transitEncryption": "ENABLED",
        "transitEncryptionPort": 2049,
        "authorizationConfig": {
          "accessPointId": "fsap-0920405ee92c0e662",
          "iam": "DISABLED"
        }
      }
    }
  ],
  "status": "ACTIVE",
  "requiresAttributes": [
    {
      "name": "ecs.capability.execution-role-awslogs"
    },
    {
      "name": "com.amazonaws.ecs.capability.ecr-auth"
    },
    {
      "name": "com.amazonaws.ecs.capability.docker-remote-api.1.28"
    },
    {
      "name": "com.amazonaws.ecs.capability.task-iam-role"
    },
    {
      "name": "ecs.capability.execution-role-ecr-pull"
    },
    {
      "name": "com.amazonaws.ecs.capability.docker-remote-api.1.18"
    },
    {
      "name": "ecs.capability.task-eni"
    },
    {
      "name": "com.amazonaws.ecs.capability.docker-remote-api.1.29"
    },
    {
      "name": "com.amazonaws.ecs.capability.logging-driver.awslogs"
    },
    {
      "name": "ecs.capability.efsAuth"
    },
    {
      "name": "com.amazonaws.ecs.capability.docker-remote-api.1.19"
    },
    {
      "name": "ecs.capability.efs"
    },
    {
      "name": "com.amazonaws.ecs.capability.docker-remote-api.1.25"
    }
  ],
  "placementConstraints": [],
  "compatibilities": [
    "EC2",
    "FARGATE"
  ],
  "requiresCompatibilities": [
    "FARGATE"
  ],
  "cpu": "512",
  "memory": "1024",
  "runtimePlatform": {
    "cpuArchitecture": "X86_64",
    "operatingSystemFamily": "LINUX"
  },
  "registeredAt": "2024-11-04T05:59:18.397Z",
  "registeredBy": "arn:aws:sts::<ACCOUNT_ID>:assumed-role/Role/joongseok.jeon",
  "tags": []
}
```

#### 3.1.1. ECS Task 접속 방법

```bash
# Session Manager 설치
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb" -o "session-manager-plugin.deb"
sudo dpkg -i session-manager-plugin.deb
```

```bash
aws ecs update-service --cluster VidCluster --service VidService --enable-execute-command
```

```bash
aws ecs execute-command \
    --cluster VidCluster \
    --task df5347efb0a54434b04af0541a0fd6cb \
    --container youtube \
    --command "/bin/bash" \
    --interactive
```

### 3.2. Code Build

`buildspec.yaml`
```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws --version
      - aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com
      - REPOSITORY_URI=<ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com/youtube
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}
  build:
    commands:
      - echo Building the Docker image...
      - docker build -t youtube .
      - docker tag youtube:latest <ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com/youtube:latest
  post_build:
    commands:
      - echo Pushing the Docker image...
      - docker push <ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com/youtube:latest
      - echo Writing image definitions file...
      - printf '[{"name":"youtube","imageUri":"%s"}]' <ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com/youtube:latest > imagedefinitions.json

artifacts:
  files: 
    - imagedefinitions.json
    - task-definition.json
    - appspec.yaml
```

`imagedefinitions.json` 예시
```json
[
  {
    "name": "youtube",
    "imageUri": "<ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com/youtube:latest"
  }
]
```

### 3.3. Code Pipeline

#### 3.3.1. Rolling Update

Code Deploy를 위해선 Code Pipeline에서 기 생성한 Source, Build를 포함하고 Deploy 설정이 필요하다.

**최소 및 최대 실행 작업**

100% min 및 200% max

기본 설정으로 배포할 경우, 롤링 업데이트 방식으로 배포가 진행된다.

1. 기존 3개의 태스크 실행
2. 새로운 3개의 태스크 실행
3. 기존 3개의 태스크 Draining

> Maximum Percent가 200%로 설정되어 현재 태스크 수의 두 배까지 실행할 수 있어, 롤링 업데이터 동안 기존 태스크와 새 태스크가 동시에 실행된다.

> 50% min 및 150% max
> Minimum Healthy Percent와 Maximum Percent 값을 조정하면, ECS가 한 번에 n개의 태스크만 종료하고 새 태스크를 시작하도록 할 수 있다.
> 1. 기존 3개의 태스크 중 하나를 종료하고 새 태스크 시작
> 2. 새로운 태스크가 정상적으로 시작되면 기존 태스크가 종료되고, 또 하나의 새로운 태스크가 시작
> 3. 기존 태스크도 새로운 태스크로 대체

#### 3.3.2. Blue/Green Update

Blue/Green 배포 방식은 Code Deploy와 연계하여 ECS 서비스가 새로운 태스크 집합을 테스트한 후 트래픽을 전환하도록 설정할 수 있다.

ECS 서비스 생성 시 Blue/Green으로 설정 후 배포하게 되면 CodeDeploy에 배포 그룹이 생성되며, Blue Green을 위해 기본적으로 두 개의 대상그룹이 생성된다.

`appspec.yaml`
```yaml
version: 0.0
Resources:
  - myEcsService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: "arn:aws:ecs:ap-northeast-2:<ACCOUNT_ID>:task-definition/VidTask"
        LoadBalancerInfo:
          ContainerName: "youtube"
          ContainerPort: 4000
```

`task-definition.json`
```json
{
  "containerDefinitions": [
    {
      "name": "youtube",
      "image": "<ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com/youtube:latest",
      "cpu": 0,
      "portMappings": [
        {
          "name": "youtube-4000-tcp",
          "containerPort": 4000,
          "hostPort": 4000,
          "protocol": "tcp",
          "appProtocol": "http"
        }
      ],
      "essential": true,
      "environment": [
        {
          "name": "MONGO_USERNAME",
          "value": "mongo"
        },
        {
          "name": "MONGO_URL",
          "value": "10.0.2.90"
        },
        {
          "name": "COOKIE_SECRET",
          "value": "youtube"
        },
        {
          "name": "MONGO_PASSWORD",
          "value": "mongo"
        }
      ],
      "mountPoints": [
        {
          "sourceVolume": "YouTube",
          "containerPath": "/app/uploads",
          "readOnly": false
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/VidTask",
          "mode": "non-blocking",
          "awslogs-create-group": "true",
          "max-buffer-size": "25m",
          "awslogs-region": "ap-northeast-2",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ],
  "family": "VidTask",
  "taskRoleArn": "arn:aws:iam::<ACCOUNT_ID>:role/ecsTaskExecutionRole",
  "executionRoleArn": "arn:aws:iam::<ACCOUNT_ID>:role/ecsTaskExecutionRole",
  "networkMode": "awsvpc",
  "volumes": [
    {
      "name": "YouTube",
      "efsVolumeConfiguration": {
        "fileSystemId": "fs-0a29dd8823cab60b5",
        "rootDirectory": "/",
        "transitEncryption": "ENABLED",
        "transitEncryptionPort": 2049,
        "authorizationConfig": {
          "accessPointId": "fsap-0920405ee92c0e662",
          "iam": "DISABLED"
        }
      }
    }
  ],
  "status": "ACTIVE",
  "requiresAttributes": [
    {
      "name": "ecs.capability.execution-role-awslogs"
    },
    {
      "name": "com.amazonaws.ecs.capability.ecr-auth"
    },
    {
      "name": "com.amazonaws.ecs.capability.docker-remote-api.1.28"
    },
    {
      "name": "com.amazonaws.ecs.capability.task-iam-role"
    },
    {
      "name": "ecs.capability.execution-role-ecr-pull"
    },
    {
      "name": "com.amazonaws.ecs.capability.docker-remote-api.1.18"
    },
    {
      "name": "ecs.capability.task-eni"
    },
    {
      "name": "com.amazonaws.ecs.capability.docker-remote-api.1.29"
    },
    {
      "name": "com.amazonaws.ecs.capability.logging-driver.awslogs"
    },
    {
      "name": "ecs.capability.efsAuth"
    },
    {
      "name": "com.amazonaws.ecs.capability.docker-remote-api.1.19"
    },
    {
      "name": "ecs.capability.efs"
    },
    {
      "name": "com.amazonaws.ecs.capability.docker-remote-api.1.25"
    }
  ],
  "compatibilities": [
    "EC2",
    "FARGATE"
  ],
  "requiresCompatibilities": [
    "FARGATE"
  ],
  "cpu": "512",
  "memory": "1024",
  "runtimePlatform": {
    "cpuArchitecture": "X86_64",
    "operatingSystemFamily": "LINUX"
  },
  "tags": [
    {
      "key": "Environment",
      "value": "Development"
    }
  ]
}
```