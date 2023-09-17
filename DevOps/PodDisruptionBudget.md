# PodDisruptionBudget

Pod의 개수는 컨트롤러가 붙어 있을 경우에는 컨트롤러 스펙에 정의된 replica 수 만큼을 항상 유지하도록 되어 있다. Pod의 수가 replica의 수를 유지하지 못하고 줄어드는 경우가 존재할 수 있다.

노드 인위적으로 줄일 경우 해당 노드의 Pod의 수가 줄어들 수 있다. replica 수에 의해서 복귀는 되겠지만, 성능을 유지하기 위해서 일정 수의 Pod를 유지해야 하거나, 안정성을 확보하기 위해서 최소 Pod를 유지해야 하는 경우에 노드의 다운은 문제가 될 수 있다.

따라서 인위적인 노드 다운 등과 같은 상황에도 항상 최소한의 Pod 수를 유지하도록 해주는 것이 PodDisruptioBudget(PDB)이다. PDB를 설정하면 서비스 중단 없이 노드 Drain 등 클러스터 관리 작업을 위해 PDB 설정이 적절히 되어 있어야 한다.

> 예시
>
> | node1 | node2 | node3 |
> | ----- | ----- | ----- |
> | pod-1 | pod-2 | pod-3 |
>
> 상기와 같은 상태에서 관리자가 node1에 대한 패치를 위해 drain을 실행한다. node1에 있는 pod-1은 다른 node로 eviction이 이루어진다.
>
> | node1 (Draining)    | node2                       | node3 |
> | ------------------- | --------------------------- | ----- |
> | pod-1 (terminating) | pod-2 <br> pod-4 (creating) | pod-3 |
>
> 이때 생성중인 Pod가 기동되는 것을 기다리지 않고, 이외의 node를 drain 시 현재 사용 가능한 Pod는 하나 밖에 남지 않기 때문에, PDB에서 정의한 Pod의 수를 충족하지 못하기 때문에, drain 작업은 중단된다.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: youtube-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: youtube
```

## Drain Node

```bash
kubectl drain node3 -ignore-daemonsets --delete-emptydir-data
```

특정 노드를 Drain 직후의 결과이다. PDB를 적용하였기 때문에, `minAvailable: 2`를 지켜 기존의 노드에서 순차적으로 Terminating 되고 다른 노드로 Scheduling 되는 것을 확인할 수 있다.

```bash
NAME                                      READY   STATUS    RESTARTS      AGE    IP                NODE
pod/youtube-deployment-5db657dc44-b5x7t   1/1     Running   0             8d     10.233.75.19      node2
pod/youtube-deployment-5db657dc44-cshdk   1/1     Running   0             8d     10.233.71.29      node3
pod/youtube-deployment-5db657dc44-kc7nb   1/1     Running   0             8d     10.233.71.48      node3
```

```bash
NAME                                      READY   STATUS    RESTARTS      AGE     IP               NODE
pod/youtube-deployment-5db657dc44-b5x7t   1/1     Running   0             8d      10.233.75.19     node2
pod/youtube-deployment-5db657dc44-cshdk   1/1     Running   0             8d      10.233.71.29     node3
pod/youtube-deployment-5db657dc44-ptpnj   1/1     Running   0             7s      10.233.102.130   node1
```

```bash
NAME                                      READY   STATUS    RESTARTS      AGE     IP               NODE
pod/youtube-deployment-5db657dc44-b5x7t   1/1     Running   0             8d      10.233.75.19     node2
pod/youtube-deployment-5db657dc44-ns84s   1/1     Running   0             14s     10.233.102.179   node1
pod/youtube-deployment-5db657dc44-ptpnj   1/1     Running   0             60s     10.233.102.130   node1
```

만약, PDB가 설정되어 있지 않다면, `node3`에 서비스되고 있는 애플리케이션은 모두 Terminating 되고 다른 노드로 Scheduling 될 것이다. 즉, 1개의 Pod만 서비스 정상 상태를 유지한다.

> Drain으로 Scheduling이 중단된 노드를 uncordon하여 Schedule이 가능하도록 복구해야 한다.

## Kubernetes Cluster Upgrade

PDB를 사용하여 서비스에 영향 없이 안전하게 재배치가 가능한 지 테스트를 하겠다.

Kubernetes의 Cluster를 업그레이드하는 전략은 하기와 같다.

1. Control Plane 업그레이드
2. Worker Node 업그레이드

Kubernetes의 Cluster 정보

```bash
NAME     STATUS   ROLES           AGE   VERSION
master   Ready    control-plane   40d   v1.26.5
node1    Ready    <none>          40d   v1.26.5
node2    Ready    <none>          40d   v1.26.5
node3    Ready    <none>          40d   v1.26.5
```

테스트 서버 정보

```bash
NAME                                     READY   STATUS    RESTARTS   AGE     IP               NODE    NOMINATED NODE   READINESS GATES
pod/youtube-deployment-ccf6db8bb-7ds6p   1/1     Running   0          8m38s   10.233.102.163   node1   <none>           <none>
pod/youtube-deployment-ccf6db8bb-g6zhn   1/1     Running   0          9m18s   10.233.71.53     node3   <none>           <none>
pod/youtube-deployment-ccf6db8bb-gpq2p   1/1     Running   0          10m     10.233.71.2      node3   <none>           <none>
pod/youtube-deployment-ccf6db8bb-n4b2x   1/1     Running   0          5m40s   10.233.75.42     node2   <none>           <none>
pod/youtube-deployment-ccf6db8bb-pxt57   1/1     Running   0          9m59s   10.233.102.146   node1   <none>           <none>

NAME                      TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)        AGE   SELECTOR
service/youtube-service   LoadBalancer   10.233.25.134   192.168.100.240   80:30491/TCP   16d   app=youtube

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                        SELECTOR
deployment.apps/youtube-deployment   5/5     5            5           16d   youtube      ghcr.io/ddung1203/youtube:5   app=youtube

NAME                                            DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES                             SELECTOR
replicaset.apps/youtube-deployment-55f4948dc8   0         0         0       16d   youtube      ghcr.io/ddung1203/youtube:latest   app=youtube,pod-template-hash=55f4948dc8
replicaset.apps/youtube-deployment-ccf6db8bb    5         5         5       10m   youtube      ghcr.io/ddung1203/youtube:5        app=youtube,pod-template-hash=ccf6db8bb
```

PDB 정보

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: youtube-pdb
spec:
  minAvailable: 3
  selector:
    matchLabels:
      app: youtube
```

> **Vegeta**
> 
> HTTP 통신을 통해 웹 서비스의 성능 및 부하 테스트를 수행하는데 사용된다. 웹 서비스의 응답 시간, 처리량 및 부하 처리 능력을 테스트할 수 있다.
> 
> ```bash
> echo "GET http://192.168.100.240/" | vegeta attack --duration=300s | tee results.bin | vegeta report
> ```


> 현재 사용중인 Kubernetes Cluster가 최신 버전이기 때문에, Cluster 업그레이드하는 순서와 같이 Worker Node의 drain과 uncordon 과정을 실시하여 테스트 하겠다.
> 
> 2분간 순차적으로 Worker Node를 drain과 uncordon을 적용 후 테스트
> 
> ```bash
> kubectl drain node1 --ignore-daemonsets
> kubectl uncordon node1
> 
> kubectl drain node2 --ignore-daemonsets
> kubectl uncordon node2
> 
> kubectl drain node3 --ignore-daemonsets
> kubectl uncordon node3
> ```

**PDB 미적용 상태**

```bash
$ vegeta report results.bin
Requests      [total, rate, throughput]         15000, 50.00, 19.12
Duration      [total, attack, wait]             5m2s, 5m0s, 430.727ms
Latencies     [min, mean, 50, 90, 95, 99, max]  1.379ms, 5.973s, 77.513ms, 28.609s, 30.001s, 30.001s, 30.023s
Bytes In      [total, mean]                     60220348, 1338.23
Bytes Out     [total, mean]                     0, 0.00
Success       [ratio]                           91.28%
Status Codes  [code:count]                      0:3922  200:41078
```

**PDB 적용 상태**

```bash
Requests      [total, rate, throughput]         15000, 50.00, 49.99
Duration      [total, attack, wait]             5m0s, 5m0s, 52.252ms
Latencies     [min, mean, 50, 90, 95, 99, max]  24.153ms, 297.726ms, 48.289ms, 654.753ms, 1.539s, 4.579s, 6.815s
Bytes In      [total, mean]                     21990000, 1466.00
Bytes Out     [total, mean]                     0, 0.00
Success       [ratio]                           100.00%
Status Codes  [code:count]                      200:15000 
```