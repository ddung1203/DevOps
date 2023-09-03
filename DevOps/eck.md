# ECK

ECK란, Elasticsearch와 Kibana를 Kubernetes 환경에서 배포하고 관리하기 위한 오픈소스 프로젝트이다.

> **EFK**는 Elasticsearch, Fluentd, Kibana 스택을 의미한다. 기존의 ELK의 Logstash를 Fluentd로 대체한 것이다.
> 
> **Fluentd**는 CNCF graduated project로, 다양한 소스에서 로그를 수집해 중앙화할 수 있는 오픈소스 데이터 콜렉터이다. 여러 로그들을 수집하고 파싱하고 포워딩할 수 있다. 이러한 특징으로 시스템마다 각각의 로깅 툴을 사용하고 각기 다른 저장 장소에 로그가 흩어져 있던 기존의 로깅 아키텍처를 Fluentd 하나로 통합할 수 있다는 장점이 있다. 특히, MSA 환경에서 이러한 장점이 더욱 장점이 된다. 
> 
> **Logstash vs Fluentd**
> 
> EFK 스택은 기존의 Logstash를 Fluentd로 대체해 사용하는 스택이다. 모두 엔터프라이즈급 로그 파이프라인 구성에 사요되는 로그 분석용 Parser 역할을 할 수 있다. 수집한 로그를 필터링, 라우팅할 수 있으며, 다양한 플러그인을 사용할 수 있다는 공통점이 있다. 
> 
> 하지만 Logstash는 자체 로그 수집 기능이 없으며 Filebeat, Metricbeat 같은 에이전트에서 수집한 로그를 받거나 다른 시스템과 통합하는 과정을 통해 수집된 로그를 받을 수 밖에 없다. 반면에 Fluentd는 자체 데몬을 통해 다양한 소스의 로그를 수집할 수 있다.
> 
> Fluentd는 CNCF 재단에 속한 프로젝트로서 클라우드 네이티브 시스템, 마이크로 서비스, Prometheus와 같은 CNCF 프로젝트와 함께 사용하기 용이하다는 점이 다르다.
> 
> Logstash는 jruby를 이용해 만들어져 JVM 위에서 돌아가지만, Fluentd는 대부분 ruby로 만들어졌으며 몇몇 퍼포먼스가 중요한 부분은 C로 구성되어 있다. 때문에 Fluentd보다는 Logstash가 약간 더 무거운 프로세스를 실행하게 된다.
> 
> 위 차이점 덕분에 보통 Logstash는 모놀리식한 시스템의 로깅 파이프라인으로 주로 사용하고, Fluentd는 마이크로 서비스 아키텍처를 사용한 시스템에서 주로 사용하기 용이하다.

## Feature

- Kubernetes Native: ECK는 Elasticsearch와 Kibana를 Kubernetes와 완벽하게 통합하며, Kubernetes의 기능과 개념을 활용하여 이들을 관리한다. 이로써 Elasticsearch 및 Kibana 클러스터의 배포, 확장, 업그레이드, 롤링 업데이트 등이 쉬워진다.
- Custom Resource Definitions(CRDs): ECK는 CRDs를 사용하여 Elasticsearch 및 Kibana 클러스터의 구성 및 구성 요소를 정의한다.

> **Kubernetes CR, CRD, Operator**
> 
> Kubernetes에서는 실제 애플리케이션 동작 및 운영에 필요한 Pod, Deployment, Service 등과 같은 기본적인 리소스 오브젝트만 제공하고 있다.
> 
> Prometheus 기준으로 확장을 한다고 가정하자.
> 
> Kubernetes 기본 오브젝트만을 기준으로 운영한다면 수집해야할 정보와 규칙들이 많아질 수록 프로메테우스의 설정이 복잡해지고, 대상 시스템에 따라서 하나 이상의 프로메테우스를 실행해서 연계하거나 각각 메트릭 수집을 분리해서 운영하는 등의 작업이 필요한 상황이 발생하게 된다. 운영자의 입장에서, 이러한 작업들이 자동화되어 관리만 할 수 있는 시스템을 원하지만, Kubernetes에서는 이런 작업을 할 방법이 없다. 이유는 Kubernetes는 상태관리 시스템이기 때문에 자동화를 위한 상태정보를 관리할 대상이 없기 때문이다.
> 
> Go 관점에서, Structure를 오브젝트 데이터 관리용으로 사용한다. 구조체에서 관리할 데이터 항목과 형식을 지정하며, 관리할 데이터를 정의한다.
> 
> 오브젝트를 만들었다고 해도 Kubernetes는 이 오브젝트를 인식하지 못하기 때문에 GVR(Group Version Resource) 형식으로 Kubernetes가 인식할 수 있도록 등록해 줘야 한다. 즉, Kubernetes에 사용자가 정의한 오브젝트에 대한 이름과 형식과 사이즈 등의 데이터 관리 정보를 GVR 기준으로 정의한 것이 CRD이다.
> 
> CRD로 Kubernetes가 인식할 수 있는 오브젝트의 정의를 등록했다면, 실제 상태 정보를 관리할 수 있는 오브젝트가 CR이다. 
> 
> CR의 인스턴스들에 변경이 발생했을 때 Kubernetes를 통해 변경 이벤트를 받아서 처리할 사용자 정의 Controller가 필요하다.
> 
> 지금까지 기술한 CRD 구성, Controller 등을 합쳐 놓은 것이 Operator이다.
> 
> *너무 어렵다... 얼핏 알거는 같지만 다시 차근차근 공부해보자!*

## Download ECK(Elastic Cloud on Kubernetes)

https://www.elastic.co/kr/downloads/elastic-cloud-kubernetes

```bash
kubectl create -f https://download.elastic.co/downloads/eck/2.9.0/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.9.0/operator.yaml
```

혹은 Helm 차트를 사용하여 ECK를 설치한다.

https://www.elastic.co/guide/en/cloud-on-k8s/master/k8s-install-helm.html

```bash
helm repo add elastic https://helm.elastic.co
helm repo update

helm install elastic-operator elastic/eck-operator -n elastic-system --create-namespace
```

> 기존 Elastic Stack(Elasticsearch, Kibana)을 설치하는 Helm chart의 경우, `8.5.1` 버전 이후로는 출시되지 않고 있다. ECK를 사용하여 Elastic Stack을 관리할 것을 권장하고 있으며, CR, CRD, Operator 등의 개념에 대해 알게된 계기가 되었다.
> 
> https://github.com/elastic/helm-charts#elastic-stack-kubernetes-helm-charts

## Deploy an Elasticsearch cluster

Operator는 Elasticsearch 클러스터의 상태를 달성하기 위해 자동으로 K8s 리소스를 만들고 관리한다. 

하기의 경우, TLS 인증서를 disabled 하였다. Self-signed한 인증서(하기의 경우 `elastic-certificates`)는 Fluentd 및 Fluent-bit에서 Elasticsearch로 log 등을 전송할 때 오류가 발생하며, 따라서 테스트의 목적으로 disabled한 상황에서 운영을 하였다.

[TLS certificates 참고](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-tls-certificates.html)

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch
spec:
  version: 8.9.1
  nodeSets:
    - name: default
      count: 1
      config:
        node.store.allow_mmap: false
  # http:
  #   tls:
  #     certificate:
  #       secretName: elastic-certificates
  http:
    tls:
      selfSignedCertificate:
        disabled: true
```

## Deploy a Kibana instance

만약, TLS 인증서를 사용하여 Kibana Web을 사용하고자 하는 경우, 하기의 주석을 제거한다. 

```yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana
spec:
  version: 8.9.1
  count: 1
  elasticsearchRef:
    name: elasticsearch
  # http:
  #   tls:
  #     certificate:
  #       secretName: elastic-certificates
```

> CRD인 Elasticsearch/Kibana는 Elasticsearch/Kibana를 Kubernetes 클러스터에서 배포하고 관리하기 위한 CR 타입이다.
> 
> Kubernetes에서는 CR 타입을 정의할 수 있으며, 이러한 CR 타입을 CRD를 통해 Kubernetes API에 추가할 수 있다. Elasticsearch/Kibana CRD는 Elasticsearch/Kibana 클러스터를 정의하고 구성하기 위한 자체 리소스를 만들 수 있도록 도와주며, Elasticsearch/Kibana 클러스터를 Kubernetes 환경에서 관리하는 데 사용된다.

## Deploy a Fluentd using Helm Chart

Fluent Helm repo 추가

```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update
```

Value 수정 및 버전 관리를 위한 Chart Download

```bash
helm pull fluent/fluentd
tar xvfz fluentd-0.4.4.tgz
mv fluentd fluentd-0.4.4
```

변경 사항들은 하기와 같다.

`values.yaml`
```yaml
variant: elasticsearch8
...
env:
  - name: K8S_NODE_NAME
    valueFrom:
      fieldRef:
        fieldPath: spec.nodeName
...
configMapConfigs:
  - fluentd-prometheus-conf
...
  04_outputs.conf: |-
    <label @OUTPUT>
      <match **>
        @type elasticsearch
        hosts "http://elastic:${PASSWORD}@elasticsearch-es-default.elastic.svc.cluster.local:9200"
        # port 9200
        # path ""
        # user elastic
        # password ${PASSWORD}
      </match>
    </label>
```

- `elasticsearch-es-default`의 DNS 작성
- elastic의 `password`는 하기 명령어로 조회 가능
- TLS 인증서를 사용한다면, 하기 `yaml` 파일 참고

```bash
kubectl get secret elasticsearch-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode; echo
```

`values.yaml`
```yaml
volumes:
  - name: elastic-certs
    secret:
      secretName: elastic-certificates
      defaultMode: 0777
...
volumeMounts:
  - name: elastic-certs
    mountPath: /etc/fluent/certs
    readOnly: true
...
  04_outputs.conf: |-
    <label @OUTPUT>
      <match **>
        @type elasticsearch
        hosts "https://elastic:${PASSWORD}@elasticsearch-es-default.efk.svc.cluster.local:9200"
        ca_file /etc/fluent/certs/ca.crt
        client_cert /etc/fluent/certs/tls.crt
        client_key /etc/fluent/certs/tls.key
        # port 9200
        # path ""
        # user elastic
        # password ${PASSWORD}
      </match>
    </label>
```

> Self-signed한 인증서의 경우 사용이 불가능하다.
> 
> `certificate verify failed (self signed certificate) (OpenSSL::SSL::SSLError) Unable to verify certificate.`


> probe endpoint는 항상 실행되어야 하는 prometheus server에 의해 활성화 된다. 만약, `configMapConfigs: fluentd-prometheus-conf`를 작성하지 않았을 경우 Pod가 Readiness Probe로 인해 종료되는 것을 확인할 수 있다.
> 
> [Git Issue 참고](https://github.com/fluent/helm-charts/issues/84#issuecomment-801792696)