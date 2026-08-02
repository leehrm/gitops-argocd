# Kiali 트래픽 그래프 읽기 가이드

이 문서는 현재 프로젝트의 Kiali `Traffic Graph`를 해석하는 기준과 확인 순서를 정리함. Kiali 그래프는 Istio proxy가 Prometheus에 기록한 일정 시간 범위의 metric을 집계한 topology이며, 개별 요청 하나의 실행 순서를 그대로 재생하는 distributed trace가 아님.

## 용어와 기본 개념

### Kiali

Istio service mesh의 topology, traffic metric, mTLS 상태와 Istio configuration을 조회하는 관리·관측 UI임. 자체적으로 traffic을 수집하거나 저장하지 않고 주로 Prometheus의 Istio metric과 Kubernetes·Istio resource를 조회함.

### Service mesh

애플리케이션 사이의 network traffic을 proxy 계층에서 전달·암호화·관측·제어하는 구조임. 현재 프로젝트에서는 Istio Gateway와 workload별 Envoy sidecar가 data plane을 구성하고 `istiod`가 proxy configuration과 workload 인증서를 관리함.

### Namespace

Kubernetes resource를 구분하는 논리적 경계임. 현재 그래프의 중심 namespace는 `argo-task-api-istio`이며 연결 관계에 따라 `istio-ingress`, `notification-service`, `redis`, `tracing`의 node도 함께 표시될 수 있음.

### Application

Kiali가 `app` 계열 label을 기준으로 관련 workload를 논리적으로 묶은 단위임. Application 수는 Pod 수나 replica 수를 의미하지 않음.

### Workload

실제 Pod를 생성하고 관리하는 Deployment 등의 실행 단위임. Kiali의 Workload graph는 Application graph보다 실제 배포 단위에 가까운 정보를 제공함.

### Service

Pod의 IP가 변경되어도 일정한 DNS 이름과 virtual IP로 workload에 접근하게 하는 Kubernetes resource임. 그래프에서 Service node와 요청을 처리한 workload node가 함께 표시될 수 있음.

### Gateway

클러스터 외부 요청을 받는 진입점임. 현재 프로젝트의 `task-api-istio-gateway-istio`는 public HTTPS를 종료하고, 허용된 요청을 비교용 Task API로 전달함.

### Sidecar

애플리케이션과 같은 Pod에서 실행되는 Envoy proxy임. 애플리케이션의 inbound·outbound traffic을 처리하고 HTTP/TCP metric, source·destination 정보와 mTLS 정보를 생성함.

### Node

그래프에 표시되는 Application, Workload, Service, Gateway 또는 외부 목적지 단위임. node 모양과 badge는 선택한 graph type과 display option에 따라 달라질 수 있으므로 이름, namespace와 우측 상세 panel을 함께 확인함.

### Edge

두 node 사이의 화살표임. 화살표 방향은 caller에서 destination으로 향하며, edge에는 request rate, byte rate, response time과 오류 상태가 표시될 수 있음.

### Inbound와 Outbound

- `Inbound`: 선택한 node로 들어오는 traffic임
- `Outbound`: 선택한 node에서 나가는 traffic임
- `Total`: 선택 범위의 inbound와 outbound를 합친 요약임

### RPS

`Requests Per Second`의 약자로 초당 요청 수임. 15분마다 한 번 발생하는 요청은 약 `0.0011 rps`이므로 화면에서 `0rps` 또는 `0.00`으로 반올림될 수 있음.

### BPS

`Bytes Per Second`의 약자로 초당 전송된 byte 수임. Redis, PostgreSQL, 외부 HTTPS처럼 Kiali가 HTTP 요청으로 해석하지 못한 TCP traffic은 주로 `bps`로 표시됨.

### Response time

요청을 보낸 뒤 응답을 받을 때까지의 시간임. edge의 `ms` 값으로 표시되며 선택한 시간 범위의 집계값임. 개별 요청의 정확한 처리 구간은 Grafana Tempo trace에서 확인함.

### HTTP status와 오류율

- `OK`: 주로 2xx 성공 응답임
- `3xx`: redirect 응답임
- `4xx`: 잘못된 요청, 존재하지 않는 경로 또는 인가 거부 등임
- `5xx`: 애플리케이션, proxy 또는 upstream 처리 오류임
- `NR`: 정상 HTTP 응답을 받지 못한 요청임

색상은 traffic 상태와 health를 빠르게 구분하는 용도임. 초록색 edge만으로 mTLS 적용 여부를 판단하지 않음.

### mTLS

통신 양쪽 proxy가 서로 workload 인증서를 확인한 뒤 traffic을 암호화하는 방식임. Kiali의 `Display`에서 `Security` badge를 활성화하면 mTLS가 관측된 edge에 자물쇠가 표시됨. 자물쇠는 Prometheus destination metric의 `connection_security_policy`를 기준으로 계산함.

### PassthroughCluster

Envoy가 Kubernetes Service 또는 Istio `ServiceEntry`로 식별하지 못한 목적지에 traffic을 전달할 때 사용하는 built-in cluster임. 현재 프로젝트에서는 외부 RDS 같은 mesh 외부 연결이 포함될 수 있음. 이 node만으로 실제 목적지가 Slack인지 RDS인지 단정하지 않음.

### Graph type

- `Service graph`: Service 중심의 높은 수준 topology임
- `App graph`: 같은 application label을 가진 대상을 묶어 표시함
- `Versioned app graph`: application과 version label을 함께 구분함
- `Workload graph`: Deployment 등 실행 workload 중심으로 표시함

전체 연결 구조는 `App graph`, 실제 배포 단위 확인은 `Workload graph`, version별 traffic 분배 확인은 `Versioned app graph`가 적합함.

### Time range와 refresh

- `Last 30m`: 현재 시점 이전 30분의 metric을 집계함
- `Every 1m`: 화면을 1분마다 새로 조회함
- `Replay`: 과거 시간 범위의 graph 상태를 다시 확인함

Kiali는 선택한 시간 범위에 실제로 기록된 Istio metric으로 graph를 구성함. traffic이 적으면 짧은 시간 범위에서 edge가 사라질 수 있음.

## 현재 프로젝트의 traffic 구조

### 외부 Gateway traffic

```text
외부 요청
  -> task-api-istio-gateway-istio
  -> argo-task-api-istio Service
  -> task-api Workload
```

`GET https://task-api-istio.lhrm-lab.com/version` 요청이 이 edge를 생성함. Kiali는 사용자에서 Cloudflare DNS와 AWS LoadBalancer까지의 인터넷 구간을 상세하게 관찰하지 않으며, Istio Gateway가 기록한 traffic부터 표시함.

### 내부 Notification traffic

```text
task-api
  -> notification-service Service
  -> notification-service Workload
  -> Slack HTTPS
```

Task가 미완료에서 완료 상태로 변경되면 Task API가 `POST /notifications/task-completed`를 호출함. Task API와 Notification Service 사이에는 sidecar와 Istio mTLS가 적용됨.

### Redis traffic

```text
task-api
  -> redis-master Service
  -> redis Workload
```

Redis 통신은 HTTP가 아닌 TCP이므로 주로 파란색 edge와 `bps`로 표시됨. HTTP path와 status code는 제공되지 않음.

### OpenTelemetry traffic

```text
task-api
  -> opentelemetry-collector Service
  -> opentelemetry-collector Workload
```

Task API가 OTLP/gRPC로 trace를 Collector에 전송하는 흐름임. Kiali는 network edge를 표시하고, trace 내부의 FastAPI·Redis·PostgreSQL span은 Grafana Tempo에서 확인함.

### Mesh 외부 traffic

```text
task-api 또는 notification-service
  -> PassthroughCluster
  -> mesh 외부 목적지
```

외부 목적지가 Istio service registry에 등록되지 않으면 구체적인 서비스 이름 대신 `PassthroughCluster`가 표시될 수 있음. 외부 HTTPS는 암호화된 상태로 sidecar를 통과하므로 HTTP가 아니라 TCP traffic으로 보일 수 있음.

## 그래프를 읽는 순서

### 1. Namespace와 시간 범위 확인

현재 프로젝트의 기본 확인값은 다음과 같음.

```text
Namespace: argo-task-api-istio
Graph type: App graph 또는 Workload graph
Time range: Last 30m 이상
Refresh: Every 1m
```

traffic generator가 15분 간격으로 실행되므로 `Last 5m`에서는 graph가 비어 보일 수 있음. 최소 `Last 30m`를 사용함.

### 2. 화살표 방향으로 caller와 destination 확인

화살표의 시작점이 요청을 보낸 caller이고 화살표 끝이 destination임. 다음 두 방향을 먼저 확인함.

```text
Gateway -> Task API
Task API -> Notification Service
```

### 3. HTTP와 TCP 구분

- `rps`, response time과 HTTP status가 보이면 HTTP 또는 gRPC traffic으로 해석함
- `bps` 중심이면 TCP traffic으로 해석함
- 외부 HTTPS는 암호화 때문에 TCP로 보일 수 있음

### 4. 요청량과 latency 확인

`0rps`만 보고 traffic이 없다고 판단하지 않음. 15분 간격 traffic은 반올림되어 `0rps`로 보일 수 있으므로 edge 존재 여부, 선택 시간과 우측 상세 panel을 함께 확인함.

Notification Service edge의 response time은 내부 HTTP 전달뿐 아니라 Notification Service가 Slack 응답을 기다린 시간을 포함할 수 있음.

### 5. 성공률과 오류 색상 확인

우측 panel에서 `% Success`, `% Error`와 status 분포를 확인함. 주황색이나 빨간색 edge가 있으면 해당 edge를 선택하여 4xx·5xx·NR 중 어떤 유형인지 구분함.

### 6. mTLS 확인

`Display -> Security` badge를 활성화하고 edge의 자물쇠를 확인함. 현재 프로젝트에서 다음 구간은 mTLS가 기대됨.

```text
Istio Gateway -> Task API sidecar
Task API sidecar -> Notification Service sidecar
```

자물쇠가 없으면 traffic이 너무 적거나 destination metric이 수집되지 않은 경우도 있으므로 `PeerAuthentication`, `AuthorizationPolicy`, Prometheus metric과 sidecar 상태를 함께 확인함.

### 7. Node 또는 edge를 선택하여 상세 확인

- node 선택: inbound·outbound 요약, health와 관련 resource를 확인함
- edge 선택: source·destination 사이의 request rate, response time, status와 mTLS 비율을 확인함
- double click: 선택한 node 중심의 detail graph로 이동함

전체 graph가 선택된 상태의 `5 apps`, `5 services`, `9 edges`는 선택 범위의 집계이며 replica 수가 아님.

## 현재 화면의 해석 예시

현재 프로젝트 graph를 다음 순서로 읽음.

1. `task-api-istio-gateway-istio -> argo-task-api-istio -> task-api` edge가 있으면 public hostname 요청이 Gateway와 Task API까지 도달했음을 의미함.
2. `task-api -> notification-service` edge가 있으면 Task 완료 후 내부 Notification API 호출이 발생했음을 의미함.
3. Notification edge가 초록색이고 우측 panel이 `100% Success`이면 선택 시간 범위에서 관측된 HTTP 요청이 정상 응답했음을 의미함.
4. `task-api -> redis-master -> redis`의 파란색 edge는 Redis TCP byte traffic을 의미함.
5. `task-api -> opentelemetry-collector` edge는 trace export traffic을 의미함.
6. `PassthroughCluster` edge는 registry에 이름이 등록되지 않은 mesh 외부 연결이 있었음을 의미함.
7. 빨간색·주황색 edge가 없다는 사실은 관측된 요청에서 높은 오류율이 없다는 의미이며, traffic이 없던 기능까지 정상임을 보장하지는 않음.

## Traffic generator를 볼 때의 주의점

Traffic generator는 15분마다 두 종류의 synthetic traffic을 만듦.

```text
외부 관측용 요청
GET public /version
  -> Istio Gateway
  -> Task API

내부 관측용 요청
Task 생성
  -> 완료 처리
  -> Notification Service
  -> Slack
  -> Task 삭제
```

두 흐름은 같은 시간대에 발생하지만 동일 HTTP request나 동일 distributed trace로 연결된 하나의 요청은 아님. Kiali graph는 선택한 시간 범위의 집계 topology이므로 서로 다른 요청의 edge를 하나의 graph에 함께 표시함.

CronJob으로 배포하면 public 요청은 cluster 내부 generator가 public LoadBalancer로 나갔다가 Gateway로 다시 들어오는 hairpin traffic임. 실제 사용자가 만든 요청은 아니지만 Gateway 이후 Kiali graph와 metric을 유지하는 용도로 사용함.

## 자주 오해하는 표시

### `0rps`이면 traffic이 없음

아님. 낮은 request rate가 반올림될 수 있음. 시간 범위를 늘리고 edge 상세와 Prometheus metric을 확인함.

### 초록색이면 mTLS임

아님. 초록색은 주로 정상 traffic 또는 health를 의미함. mTLS는 Security badge의 자물쇠와 edge 상세로 확인함.

### 그래프에서 node가 연결되어 있으면 같은 요청임

아님. Kiali graph는 시간 범위의 집계 topology임. 요청 하나의 전체 실행 순서는 Tempo distributed trace로 확인함.

### `5 apps`이면 Pod가 5개임

아님. Application label 기준의 논리적 집계임. Pod와 replica는 Workloads 화면 또는 `kubectl get pods`로 확인함.

### `PassthroughCluster`이면 오류임

항상 오류는 아님. `ALLOW_ANY` 환경에서 등록되지 않은 외부 목적지로 정상 전달될 때도 나타남. 예상하지 않은 traffic이면 목적지와 `ServiceEntry` 누락 여부를 확인함.

### 그래프가 비어 있으면 서비스 장애임

반드시 그렇지 않음. 선택 시간에 traffic이 없거나, sidecar·Prometheus scrape·namespace 선택에 문제가 있을 수 있음.

## 문제 상황별 확인 순서

| 현상 | 우선 확인할 내용 |
|---|---|
| graph가 비어 있음 | Namespace, Last 30m 이상, 실제 요청 발생, sidecar `2/2`, Prometheus target |
| Gateway edge가 없음 | public `/version` 응답, HTTPRoute, Gateway 상태, `istio-ingress` 포함 여부 |
| Notification edge가 없음 | Task 완료 발생, Notification Service log, Task API notifier 설정 |
| edge가 4xx | HTTPRoute path, AuthorizationPolicy, method와 principal |
| edge가 5xx | destination workload log, readiness, upstream dependency |
| mTLS 자물쇠가 없음 | Security badge, destination telemetry, PeerAuthentication, sidecar injection |
| TCP node가 분리되어 보임 | 양쪽 sidecar와 mTLS 적용 여부, protocol 인식 여부 |
| PassthroughCluster가 많음 | 외부 RDS·Slack 등 예상 목적지, ServiceEntry 필요성, 미등록·오류 routing |

## Kiali와 Grafana를 함께 사용하는 기준

```text
Kiali
  -> 어느 서비스 사이에서 오류나 latency가 발생했는지 확인함

Grafana Tempo
  -> 특정 요청의 FastAPI, Redis, PostgreSQL span을 확인함

Grafana Loki
  -> 동일 시간대 또는 trace ID의 application log를 확인함

Prometheus/Grafana dashboard
  -> request rate, error ratio, resource usage의 시간 변화를 확인함
```

Kiali에서 문제 edge를 찾은 뒤 Grafana trace와 log로 원인을 좁히는 순서가 적합함.

## 참고 자료

- [Kiali topology와 graph](https://kiali.io/docs/features/topology/)
- [Kiali graph FAQ](https://kiali.io/docs/faq/graph/)
- [Kiali mTLS 표시](https://kiali.io/docs/features/security/)
