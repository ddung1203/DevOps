# Kubernetes API Server Security

## Service Account

### Service Account Resource

각 파드는 하나의 서비스와 연계되지만 여러 파드가 같은 서비스 어카운트를 사용할 수 있다.

파드 매니페스트에 서비스 어카운트의 이름을 지정해 파드에 서비스 어카운트를 할당할 수 있다. 명시적으로 할당하지 않으면 파드는 네임스페이스에 있는 default 서비스 어카운트를 사용한다.

파드에 서로 다른 서비스 어카운트를 할당하면 각 파드가 액세스할 수 있는 리소스를 제어할 수 있다. API 서버가 인증 토큰이 있는 요청을 수신하면, API 서버는 토큰을 사용해 요청을 보낸 클라이언트를 인증한 당므 관련 서비스 어카운트가 요청된 작업을 수행할 수 있는지 여부를 결정한다.

클러스터 보안을 위해, 모든 파드에 default 서비스 어카운트를 사용하는 거싱 아닌 새롭게 서비스 어카운트를 생성한다. 클러스터의 메타데이터를 읽을 필요가 없는 파드는 클러스터에 배포된 리소스를 검색하거나 수정할 수 없는 제한된 계정으로 실행해야 한다. 리소스의 메타데이터를 검색해야 하는 파드는 해당 오브젝트의 메타데이터만 읽을 수 있는 서비스 어카운트로 실행해야 하며, 오브젝트를 수정해야 하는 파드는 API 오브젝트를 수정할 수 있는 고유한 서비스 어카운트로 실행해야 한다.

**사용자 정의 서비스어카운트를 사용하는 파드**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  serviceAccountName: foo
  containers:
  - name: myapp
    image: ddung1203/demo:v1
```

## RBAC로 클러스터 보안

