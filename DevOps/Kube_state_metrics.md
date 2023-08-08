# Kubernetes State Metrics

Kubernetes를 사용하면서 운영 관점에서 다양한 메트릭 정보를 수집해 모니터링 하거나 특정 상황(Pod 장애 등)에 알림을 발송해야 하는 Needs가 있다.

이런 경우, Kubernetes 클러스터 워커노드로 사용중인 서버의 CPU, Memory, Disk 뿐만 아니라 Kubernetes 클러스터 내부의 Pod가 사용중인 리소스 매트릭과 네트워크 I/O, Deployments 개수, Pod 개수 등의 다양한 정보를 수집해야 한다.

상기와 같은 Metric 정보를 수집해서 Metric 데이터를 `/metrics` 엔드포인트로 제공한다.

[Kubernetes의 kube-state-metrics](https://github.com/kubernetes/kube-state-metrics)

위 서비스는 Prometheus를 이용하여 Metric 데이터를 수집하며, [Prometheus-stack](./Prometheus_Grafana.md) 배포 시 설치됨을 확인할 수 있다. `kube-state-metrics`가 해결해 줄 질문들의 예시는 하기와 같다.

- Pod
  - 클러스터에 몇 개의 파드가 배포되었는가?
  - 대기 중인 파드는 몇 개인가?
  - 파드 요청을 처리할 수 있을 만큼의 충분한 리소스가 있는가?
- Deployment
  - 의도한 상태에서 수행중인 Pod는 몇 개인가?
  - 몇 개의 Replica가 가용한가?
  - 어떤 Deployment가 업데이트되었는가?
- Node
  - Worker Node의 상태는 어떠한가?
  - 클러스터에서 할당할 수 있는 CPU 코어는 몇 개인가?
  - 스케쥴되지 않는 노드가 있는가?
- Job
  - Job이 언제 시작되었는가?
  - Job이 언제 완료되었는가?
  - 몇 개의 Job이 실패했는가?

이외의 Metric은 상기의 GitHub 내에서 찾을 수 있다.
