# Kaniko

Kaniko는 오픈소스 프로젝트로, Docker Image 형태로 제공된다.

https://github.com/GoogleContainerTools/kaniko

Kubernetes에 Pod로 Docker만을 설치해 빌드용 컨테이너를 만들어 리소스를 할당하게 되는 경우(DinD), 보안적 이슈로 권장을 하지 않는다.

하지만, Kaniko는 내부에서 Docker Daemon 없이 Dockerfile을 빌드할 수 있는 엔진이 구현되어 있므여 이로 인해 클러스터 내에서도 권한 문제 없이 안전하고 빠르게 Docker Build를 수행할 수 있다.

[Docker in Docker 문제점](https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/)

실행 단계

1. 빌드 할 Dockerfile 위치 정보 설정
2. 빌드 후 이미지를 Push 할 레지스트리 정보 설정
3. Kaniko 이미지 실행
4. Docker 이미지 Build 및 Push

## How to use

1. Dockerfile ( --dockerfile ) : Dockerfile
2. Context ( --context ) : 이미지를 빌드하는 데 사용하는 디렉토리를 참조하는 Docker 의 빌드 컨텍스트와 유사
3. Destination ( --destination ) : 이미지를 Push 하기 위한 Container Registry

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kaniko
spec:
  containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:debug
      args:
        - "--dockerfile=Dockerfile"
        - "--context=git://<GIT_URL>"
        - "--destination=<DOCKER_URL>"
      volumeMounts:
        - name: kaniko-secret
          mountPath: /kaniko/.docker/
  restartPolicy: Never
  volumes:
    - name: kaniko-secret
      secret:
        secretName: regsecret
        items:
          - key: .dockerconfigjson
            path: config.json
```

상기 yaml을 사용 전, `ghcr.io`에 push하기 위한 secret을 생성 후, `volumeMounts`로 credentials을 사용한다.
