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
  name: myweb-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: myweb
```
