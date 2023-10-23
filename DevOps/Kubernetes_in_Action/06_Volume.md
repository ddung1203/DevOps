# Volume

## 워커 노드 파일시스템의 파일 접근

대부분의 파드는 호스트 노드를 인식하지 못하므로 노드의 파일시스템에 있는 어떤 파일에도 접근하면 안된다. 그러나 특정 시스템 레벨의 파드(**데몬셋**)는 노드의 파일을 읽거나 파일시스템을 통해 노드 디바이스를 접근하기 위해 노드의 파일시스템을 사용해야 한다. 쿠버네티스는 hostPath 볼륨으로 가능케 한다.

### hostPath 볼륨

hostPath 볼륨은 노드 파일시스템의 특정 파일이나 디렉토리를 가리킨다. 동일 노드에 실행 중인 파드가 hostPath 볼륨의 동일 결고를 사용 중이면 동일한 파일이 표시된다.

hostPath 볼륨을 DB의 데이터 디렉토리를 저장할 위치로 사용하는 것이 아니다.

### hostPath를 사용하는 Fluentd 검사

파드가 노드의 `/var/log`와 `/var/lib/docker/containers` 디렉토리에 접근하기 위해 hostPath 두 개를 사용한다. hostPath는 노드의 시스템 파일에 읽기/쓰기를 하는 경우에만 hostPath 볼륨을 사용하며, 여러 파드에 걸쳐 데이터를 유지하기 위해서는 사용하지 않는다.

`kubectl describe pod fluentd-2xkhb`
```bash
Name:             fluentd-2xkhb
Namespace:        elastic
...
Volumes:
  varlog:
    Type:          HostPath (bare host directory volume)
    Path:          /var/log
    HostPathType:  
  varlibdockercontainers:
    Type:          HostPath (bare host directory volume)
    Path:          /var/lib/docker/containers
    HostPathType:  
```
