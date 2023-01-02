# ArgoCD 배포 전략

## ArgoCD를 이용한 Canary 배포

### Blue/Green 배포란?

> Blue-Green 배포는 애플리케이션 또는 마이크로서비스의 이전 버전에 있던 사용자 트래픽을 이전 버전과 거의 동일한 새 버전으로 점진적으로 이전하는 애플리케이션 릴리스 모델입니다. 이때 두 버전 모두 프로덕션 환경에서 실행 상태를 유지합니다.
> 

### Canary 배포란?

> Canary 배포는 기존에 배포된 서비스에 신규 서비스를 한꺼번에 배포/교체를 진행하지 않고 소량의 Pod만 일시적으로 배포하는 방식입니다.
> 
> 
> Canary 배포를 함으로서, 신규 서비스의 영향도나 반응 체크등을 할 수 있고 문제 시 소량 배포된 Pod만 정리하면 되기 때문에 편의성이 있습니다.
> 

먼저, ArgoCD에서 Blue/Green을 사용하기 위해선 Plugin을 추가해야 합니다.

Argo Rollouts

Github : [github.com/argoproj/argo-rollouts](https://github.com/argoproj/argo-rollouts)

Document : [argoproj.github.io/argo-rollouts](https://argoproj.github.io/argo-rollouts/)

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://raw.githubusercontent.com/argoproj/argo-rollouts/master/manifests/install.yaml
```

```bash
strategy:
    canary:
      maxSurge: "25%"
      maxUnavailable: 0
      steps:
      - setWeight: 25
      - pause: { duration: 10m }
```

maxSurge는 배포되는 Pod의 비율을 뜻하고, maxUnavailable는 배포될 때 Unavailable되도 되는 Pod의 수를 뜻합니다.

steps에서 setWeight는 Weight 값을 주어 트래픽을 어느 정도 인가하는지에 대한 옵션입니다.

pause는 Blue/Green 때처럼 AutoPromotion Time을 뜻합니다.

아래와 같이 시간을 지정할 수 있습니다.

```
- pause: { duration: 10 }  # 10초
- pause: { duration: 10s } # 10초
- pause: { duration: 10m } # 10분
- pause: { duration: 10h } # 10시간
- pause: { duration: -10 } # 잘못된 옵션
- pause: {}                # Auto Promotion 옵션 비활성화
```