# ConfigMap & Secret

## 컨테이너에 CLI 인자 전달

### 도커에서 명령어와 인자 정의

- ENTRYPOINT: 컨테이너가 시작될 때 호출될 명령어 정의
- CMD: ENTRYPOINT에 전달되는 인자를 정의

CMD 명령어를 사용해 이미지가 실행될 때 실행할 명령어를 지정할 수 있지만, 올바른 방법은 ENTRYPOINT 명령어로 실행하고 기본 인자를 정의하려는 경우에만 CMD를 지정하는 것이다. 그러면 아무런 인자도 정의하지 않고 이미지를 실행할 수 있다.

**Shell과 exec 형식 간의 차이점**

두 명령어는 두 가지 서로 다른 형식을 지원한다.

- `shell` 형식 - 예: ENTRYPOINT node app.js
- `exec` 형식 - 예: ENTRYPOINT ["node", "app.js"]

차이점은 내부에서 정의된 명령을 Shell로 호출하는지 여부다.

```Dockerfile
ENTRYPOINT ["node", "app.js"]
```

상기와 같이 작성하면 컨테이너 내부에서 node 프로세스를 직접 실행한다. 프로세스 목록을 나열해 직접 실행된 것을 확인할 수 있다.

`ENTRYPOINT ["npm", "start"]`
```bash
$ docker exec f3bf252cbeb0 ps x
    PID TTY      STAT   TIME COMMAND
      1 ?        Ssl    0:00 npm start
     29 ?        Rs     0:00 ps x
```

`ENTRYPOINT npm start`
```bash
$ docker exec c5603870c789 ps x
    PID TTY      STAT   TIME COMMAND
      1 ?        Ss     0:00 /bin/sh -c npm start
      7 ?        Sl     0:00 npm start
     30 ?        Rs     0:00 ps x
```

상기와 같이, 메인 프로세스(PID 1)는 node 프로세스가 아닌 shell 프로세스다. 노드 프로세스(PID 7)는 shell에서 시작된다. shell 프로세스는 불필요하므로 ENTRYPOINT 명령에서 exec 형식을 사용해 실행한다.

따라서, 컨테이너를 실행할 때 인자를 전달해 재정의 할 경우에만 `CMD`를 사용하도록 한다.

### 쿠버네티스에서 명령과 인자 재정의

쿠버네티스에서 컨테이너를 정의할 때, ENTRYPOINT와 CMD 둘 다 재정의할 수 있다. 그러기 위해 하기와 같이 컨테이너 정의 안이 command와 args 속성을 지정한다.

```yaml
kind: Pod
spec:
  containers:
  - image: image
    command: ["/bin/sh"]
    args: ["arg1", "arg2", "arg3"]
```

대부분 사용자 정의 인자만 지정하고 명령을 재정의하는 경우는 거의 없다.

| Docker | Kubernetes | 설명 |
|---|---|---|
| ENTRYPOINT | command | 컨테이너 안에서 실행되는 실행파일 |
| CMD | args | 실행파일에 전달되는 인자 |

인자를 지정하는 것은 CLI로 설정 옵션을 컨테이너에 전달하는 방법이며, 환경변수를 이용하여 설정이 가능하다.

## 컨테이너의 환경변수 설정

환경변수는 파드 레벨이 아닌 컨테이너 정의 안에 설정한다.

```yaml
kind: Pod
spec:
  containers:
  - image: image
  env:
  - name: INTERVAL
    value: "30"
  name: html-generator
```

> 각 컨테이너를 설정할 때, 쿠버네티스는 자동으로 동일한 네임스페이스 안에 있는 각 서비스에 환경변수를 노출한다. 이러한 환경변수는 기본적으로 auto-injected 설정이다.

## ConfigMap

기본적인 ConfigMap의 정리는 [GitHub - TIL, ConfigMap](https://github.com/ddung1203/TIL/blob/main/k8s/11_ConfigMap_%26_Secret.md#configmap) 페이지에 정리하였다.

### 특정 파일에 마운트된 컨피그맵 항목을 가진 파드

```yaml
spec:
  containers:
  - image: image
    volumeMounts:
    - name: myvolume
      mountPath: /etc/config.conf
      subPath: myconfig.conf
```

전체 볼륨을 마운트 하는 대신에 일부만을 마운트할 수 있다. 하지만 개별 파일을 마운트하는 이 방법은 파일 업데이트와 관련해 결함을 가지고 있다.

### 컨피그맵 볼륨 안에 있는 파일 권한 설정

기본적으로 컨피그맵 볼륨의 파일 권한은 644로 설정된다. 하기와 같이 파일 권한을 변경할 수 있다.

```yaml
volumes:
- name: config
  configMap:
    name: config
    defaultMode: "6600"
```

### 애플리케이션을 재시작하지 않고 애플리케이션 설정 업데이트

컨피그맵을 업데이트하면, 이를 참조하는 모든 볼륨의 파일이 업데이트된다. 그런 다음 변경됐음을 감지하고 다시 로드하는 것은 프로세스에 달려 있다.

만약 전체 볼륨 대신 단일 파일을 컨테이너에 마운트한 경우 파일이 업데이트되지 않는다. 개별 파일을 추가하고 원본 컨피그맵을 업데이트할 때 파일을 업데이트해야 하는 경우의 해결 방법은, 전체 볼륨을 다른 디렉토리에 마운트한 다음 해당 파일을 가리키는 심볼릭 링크를 생성하는 것이다. 컨테이너 이미지에서 심볼릭 링크를 만들거나, 컨테이너를 시작할 때 심볼릭 링크를 만들 수 있다.

## Secret

기본적인 Secret의 정리는 [GitHub - TIL, Secret](https://github.com/ddung1203/TIL/blob/main/k8s/11_ConfigMap_%26_Secret.md#secret) 페이지에 정리하였다.

기본적으로 etcd에는 시크릿을 암호화되지 않은 형식으로 저장하므로, 시크릿에 저장한 민감한 데이터를 보호하려면 마스터 노드를 보호하는 것이 필요하다. 또한 파드를 만들 수 있는 사람은 누구나 시크릿을 파드에 마운트하고 민감한 데이터에 접근이 가능하기 때문에, 권한 없는 사용자가 API 서버를 이용하지 못하게 하는 것도 포함된다.

### 환경변수로 시크릿 항목 노출

`secretKeyRef`를 사용해 환경변수로 시크릿을 노출한다. 하지만, 이 기능을 사용하는 것이 가장 좋은 방법은 아니다. 애플리케이션은 일반적으로 오류 보고서에 환경변수를 기록하거나 시작하면서 로그에 환경변수를 남겨 의도치 않게 시크릿을 노출할 수 있다. 또한 자식 프로세스는 상위 프로세스의 모든 환경변수를 상속받는데, 만약 애플리케이션이 타사 바이너리를 실행할 경우 시크릿 데이터를 어떻게 사용하는지 알 수 있는 방법이 없다.

따라서 안전을 위해서는 시크릿을 노출할 때는 항상 secret 볼륨을 사용한다.

### 기본 토큰 시크릿

모든 파드에는 secret 볼륨이 자동으로 연결돼 있다.

하기와 같이 `kubectl describe`로 secret 볼륨이 마운트된 것을 보여준다.

```bash
$ kubectl describe pod/dnsutils     
Name:             dnsutils
Namespace:        default
...
Containers:
  dnsutils:
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-kxt4s (ro)
```

시크릿이 갖고 있는 ca.crt, namespace, token은 파드 안에서 쿠버네티스 API 서버와 통신할 때 필요한 모든 것을 나타낸다. 이상적으로는 애플리케이션이 완전히 쿠버네티스를 인지하지 않도록 하고 싶지만, 쿠버네티스와 직접 대화하는 방법 외에 다른 대안이 없으면 secret 볼륨을 통해 제공된 파일을 사용한다.

> 기본적으로 `default-token`은 모든 컨테이너에 마운트되지만, 파드 스펙 안에 `automountService-AccountToken: false`로 지정하거나 파드가 사용하는 서비스 어카운트를 `false`로 지정해 비활성화할 수 있다.