# AWS Code Series

## AWS Code Series를 이용한 자동화 구성

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

## Requirements

`ECS`, `ECR`, `EFS`, `Mongo`

## Deploy

Code Build, Code Deploy, Code Pipeline을 테스트하기 위해 다음을 배포한다.

- `ECS`
- `ECR`
- `EFS`: [EFS 참고](../images/efs.png)
- `Mongo`

### ECR Push

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
```

### ECS Deploy

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

#### ECS Task 접속 방법

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

### Code Build

