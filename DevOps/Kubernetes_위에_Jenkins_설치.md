# Kubernetes 위에 Jenkins 설치하기

`jenkins-master.yaml`

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: jenkins
spec:
  serviceName: jenkins
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      serviceAccountName: jenkins
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts
        ports:
          - name: http-port
            containerPort: 8080
          - name: jnlp-port
            containerPort: 50000
        volumeMounts:
          - name: jenkins-vol
            mountPath: /var/jenkins_home
      volumes:
        - name: jenkins-vol
          hostPath:
            path: /jenkins-data
            type: Directory
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins
spec:
  type: NodePort
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: jenkins
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins-jnlp
spec:
  type: ClusterIP
  ports:
    - port: 50000
      targetPort: 50000
  selector:
    app: jenkins
```

`jenkins-rbac.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: jenkins
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get","list","watch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
- apiGroups: [""]
  resources: ["events"]
  verbs: ["get", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: jenkins
subjects:
- kind: ServiceAccount
  name: jenkins
```

테스트 환경이므로 Jenkins Volume을 hostPath로 설정하였다. 위 Pod이 생성될 Node의 '/jenkins-data' 디렉토리에 'chmod 0777' 명령어를 수행해야 정상적으로 진행할 수 있다. 실제 운영 환경에선 반드시 Persistent Volume(EBS, GlusterFS 등)을 통해 설정 및 빌드 이력 등을 잃어버리는 불상사를 피해야 한다. 그 외에 Health Check 등의 Probe 설정 등의 부가적인 요소들이 필요하다.

서비스를 통해 외부에서 NodePort로 웹에 접근할 수 있으며, Remote Agent는 50000번 포트로 연결할 수 있다. 그리고 k8s 클러스터에서 빌드를 위한 Remote Agent용 Pod을 제공하기 때문에 해당 리소스 관련 동작에 대한 권한이 필요하다.