# StatefulSet

레플리카셋은 하나의 파드 템플릿에서 여러 개의 파드 레플리카를 생성한다. 레플리카는 이름과 IP 주소를 제외하면 서로 동일하다. 파드 템플릿이 특정 PVC를 참조하는 볼륨을 포함하면 레플리카셋의 모든 레플리카는 정확히 동일한 PVC를 사용할 것이고 클레임에 바인딩된 동일한 PV를 사용하게 된다.

각 인스턴스가 별도의 스토리지를 갖기 위해선 스테이트풀셋을 사용한다. 스테이트풀셋은 애플리케이션의 인스턴스가 각각 이름과 상태를 가지며 개별적으로 취급돼야 하는 애플리케이션에 알맞게 만들어졌다.

### 거버닝 서비스

일반 파드와 달리 스테이트풀 파드는 때때로 호스트 이름을 통해 다뤄져야 할 필요가 있다. 스테이트리스 파드는 보통 그럴 필요가 없고, 각 스테이트리스 파드는 다른 스테이트리스 파드와 동일하다. 한 개가 필요하면 그중 어느 것이라도 선택하면 된다. 하지만 스테이트풀 파드는 각각 서로 다른 상태를 가지므로 그룹의 특정 파드에서 동작하기를 원한다.

상기와 같은 이유로 스테이트풀셋은 거버닝 헤드리스 서비스를 생성해서 각 파드에게 실제 네트워크 아이덴티티를 제공해야 한다. 이 서비스를 통해 각 파드는 자체 DNS 엔트리를 가지며 클러스터의 피어 혹은 클러스터의 다른 클라이언트가 호스트 이름을 통해 파드의 주소를 지정할 수 있다. 예를 들어 `mongo` 네임스페이스에 속하는 `mongo-service`라는 이름의 거버닝 서비스가 있고 파드의 이름이 `mongo-0`이라면, 이 파드는 `mongo-0.mongo-service.mongo.svc.cluster.local`이라는 FQDN을 통해 접근할 수 있다. 물론 레플리카셋으로 관리되는 파드에서는 불가능하다.

또한 `mongo-service.mongo.svc.cluster.local` 도메인의 SRV 레코드를 조회해 모든 스테이트풀셋의 파드 이름을 찾는 목적으로 DNS를 사용할 수 있다.

스테이트풀셋의 거버닝 서비스는 헤드리스여야 한다. `ClusterIP: None`으로 설정하면 헤드리스 서비스가 되며, 이로써 파드 간 피어 디스커버리를 사용할 수 있다.

```bash
root@dnsutils:/# nslookup mongo-service.mongo.svc.cluster.local
Name:	mongo-service.mongo.svc.cluster.local
Address: 10.233.71.25
Name:	mongo-service.mongo.svc.cluster.local
Address: 10.233.102.153
Name:	mongo-service.mongo.svc.cluster.local
Address: 10.233.75.13

root@dnsutils:/# nslookup mongo-statefulset-0.mongo-service.mongo.svc.cluster.local
Name:	mongo-statefulset-0.mongo-service.mongo.svc.cluster.local
Address: 10.233.102.153

root@dnsutils:/# nslookup youtube-service.youtube.svc.cluster.local
Name:	youtube-service.youtube.svc.cluster.local
Address: 10.233.25.134

root@dnsutils:/# nslookup youtube-deployment-ccf6db8bb-cs74x.youtube-service.youtube.svc.cluster.local
** server can't find youtube-deployment-ccf6db8bb-cs74x.youtube-service.youtube.svc.cluster.local: NXDOMAIN
```

## 스테이트풀셋의 피어 디스커버리

클러스터된 애플리케이션의 중요한 요구사항은 피어 디스커버리(클러스터의 다른 멤버를 찾는 기능)이다. 스테이트풀셋의 각 멤버는 모든 다른 멤버를 쉽게 찾을 수 있어야 한다.

### SRV 레코드

SRV 레코드는 특정 서비스를 제공하는 서버의 호스트 이름과 포트를 가리키는 데 사용된다. 쿠버네티스는 헤드리스 서비스를 뒷받침하는 파드의 호스트 이름을 가리키도록 SRV 레코드를 생성한다.

새 임시 파드 내부에서 `dig`를 실행해 스테이트풀 파드의 SRV 레코드를 조회할 수 있다.

```bash
root@dnsutils:/# dig SRV mongo-service.mongo.svc.cluster.local

; <<>> DiG 9.9.5-3ubuntu0.2-Ubuntu <<>> SRV mongo-service.mongo.svc.cluster.local
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 40186
;; flags: qr aa rd; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 4
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;mongo-service.mongo.svc.cluster.local. IN SRV

;; ANSWER SECTION:
mongo-service.mongo.svc.cluster.local. 5 IN SRV	0 33 27017 mongo-statefulset-1.mongo-service.mongo.svc.cluster.local.
mongo-service.mongo.svc.cluster.local. 5 IN SRV	0 33 27017 mongo-statefulset-2.mongo-service.mongo.svc.cluster.local.
mongo-service.mongo.svc.cluster.local. 5 IN SRV	0 33 27017 mongo-statefulset-0.mongo-service.mongo.svc.cluster.local.

;; ADDITIONAL SECTION:
mongo-statefulset-2.mongo-service.mongo.svc.cluster.local. 5 IN	A 10.233.71.25
mongo-statefulset-1.mongo-service.mongo.svc.cluster.local. 5 IN	A 10.233.75.13
mongo-statefulset-0.mongo-service.mongo.svc.cluster.local. 5 IN	A 10.233.102.153

;; Query time: 7 msec
;; SERVER: 169.254.25.10#53(169.254.25.10)
;; WHEN: Tue Oct 24 12:01:48 UTC 2023
;; MSG SIZE  rcvd: 627
```

`ANSWER SECTION`에는 헤드리스 서비스를 뒷받침하는 세 개의 파드를 가리키는 세 개의 SRV 레코드를 보여준다. 또한 각 파드는 `ADDITIONAL SECTION`에 표시된 것처럼 자체 A 레코드를 가진다.

파드가 스테이트풀셋의 다른 모든 파드의 목록을 가져오려면 SRV DNS 룩업을 수행하기면 하면 된다.

이를 통해, 데이터 저장소를 클러스터화 할 수 있으며, 서로 간에 커뮤니케이션 설정을 통해 완전히 다른 저장소로 독립적으로 실행하도록 할 수 있다.