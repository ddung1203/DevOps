# Pod에서 imagePullSecrets 설정

Container 테스트 시 Container Image Registry를 `hub.docker.com`을 이용했다. 하지만 `GitHub Container Registry`는 이미지 Push 시 기본적으로 private registry로 설정된다.

만약 Kubernetes에 Image를 배포할 때 private registry의 경우 imagePullSecrets을 설정해야 한다.

Secret 생성

```bash
kubectl create secret docker-registry regsecret -n test \
    --docker-server=ghcr.io                             \
    --docker-username=<USERNAME>                        \
    --docker-password=<ACCRESS_TOKEN>                   \
    --docker-email=<EMAIL>
```

Deployment 예시

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: realmytrip
  namespace: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: realmytrip
  template:
    metadata:
      labels:
        app: realmytrip
    spec:
      containers:
        - name: realmytrip
          image: ghcr.io/ddung1203/realmytrip:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 3000
      imagePullSecrets:
        - name: regsecret
```

상기와 같이 설정 시 Private Image를 가져올 수 있다. 하지만 `test` Namespace에 많은 workload들이 private repository를 호출한다면 매번 설정이 어렵다.

## Add Image Pull Secret to Service Account

```bash
kubectl patch sa default -p '{"imagePullSecrets": [{"name": "regsecret"}]}' -n test
```

혹은

```bash
kubectl get sa default -n test -o yaml > ./sa.yaml
```

`sa.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
  namespace: test
secrets:
  - name: default-token-uudge
imagePullSecrets:
  - name: regsecret
```

```bash
kubectl replace sa default -f ./sa.yaml -n test
```

이제 Pod나 Deployment 등 default Service Account를 이용하여 생성 시 자동으로 `spec.imagePullSecrets`를 추가한다.
