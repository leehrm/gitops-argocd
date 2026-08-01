# Istio·Kiali 1차 구현 및 변경 설명서

이 문서는 이번 1차 작업에서 추가한 구조를 두 단계로 설명한다.

1. 먼저 비유를 사용해 전체 역할을 쉽게 설명한다.
2. 이어서 실제 Kubernetes·Istio 리소스와 동작을 기술적으로 정확하게 설명한다.

처음 보는 전문 용어는 문서 마지막의 `용어 사전`에서 확인할 수 있다.

## 한 문장으로 설명

쉽게 설명하면, 기존 Traefik 길은 그대로 두고 똑같은 Task API 복사본으로 가는 **새 Istio 길**을 하나 만들었다. Kiali는 새 길에서 요청이 어디를 지나가는지 보여주는 교통 지도다.

기술적으로는 기존 Traefik 기반 ingress 경로와 분리된 Istio Gateway API 경로를 추가했다. 비교용 Task API Pod에는 Envoy sidecar를 주입하고, 기존 Prometheus가 Istio telemetry metric을 수집하도록 구성했다. Kiali는 Prometheus를 조회해 Gateway와 workload 사이의 요청량, 오류율, 지연시간, mTLS 상태를 시각화한다.

## 바뀌기 전과 후

기존 길은 바뀌지 않는다.

```text
인터넷
  └─ 기존 AWS LoadBalancer
       └─ Traefik
            └─ 기존 Task API
```

비교할 새 길이 옆에 생긴다.

```text
인터넷
  └─ 새 AWS LoadBalancer (HTTP 80)
       └─ Istio Gateway의 Envoy
            └─ 비교용 Task API의 Envoy sidecar
                 └─ 비교용 Task API

Envoy들이 교통량 숫자를 기록
  └─ 기존 Prometheus가 숫자를 수집
       └─ Kiali가 지도로 표시
```

Kiali 그래프는 인터넷이나 AWS LoadBalancer 자체를 그리지 않는다. 그래프에서 확인할 핵심 구간은 다음과 같다.

```text
Istio Gateway → argo-task-api-istio
```

기술적인 실제 요청 경로는 다음과 같다.

```text
AWS LoadBalancer Service
  → Istio가 관리하는 Gateway Envoy Pod
  → HTTPRoute 규칙
  → argo-task-api-istio ClusterIP Service:8000
  → Task API Pod의 Envoy sidecar
  → api container:8000
```

Prometheus는 Gateway와 sidecar Envoy가 노출하는 metric endpoint를 수집한다. Kiali는 이 metric을 조회하므로 AWS LoadBalancer나 인터넷 client를 별도 graph node로 표현하지 않고 mesh 내부의 Gateway와 workload부터 보여 준다.

## 이번에 일부러 하지 않은 것

- 기존 Traefik과 기존 `argo-task-api`는 수정하지 않았다.
- DNS와 HTTPS 인증서는 붙이지 않았다. 1차는 LoadBalancer 주소로 HTTP 테스트를 한다.
- Kiali를 인터넷에 공개하지 않았다. `kubectl port-forward`로만 접속한다.
- `/tasks` 같은 쓰기 API는 공개하지 않았다. `/version`, `/healthz`, `/readyz`만 허용한다.
- `PeerAuthentication STRICT` 정책과 내부 서비스 간 호출은 2차 작업으로 남겼다.

## 파일별 변경 설명

### 1. `clusters/dev/applications/gateway-api-crds.yaml`

쉽게 설명하면, Kubernetes가 `Gateway`와 `HTTPRoute`라는 새 단어를 알아듣게 하는 **사전**이다.

기술적으로는 Gateway API의 cluster-scoped CRD를 Argo CD Application으로 설치한다.

- 공식 Gateway API `v1.5.1`의 standard CRD를 설치한다.
- source를 `kubernetes-sigs/gateway-api` 저장소의 `config/crd/standard` 경로로 고정한다.
- 큰 CRD도 안전하게 저장되도록 `ServerSideApply=true`를 사용한다.
- 다른 Istio 리소스보다 먼저 Application 객체가 생성되도록 sync wave를 `-5`로 둔다.

### 2. `clusters/dev/applications/istio-base.yaml`

쉽게 설명하면, Istio가 사용할 표지판과 규칙을 먼저 깔아 주는 **도로 기초 공사**다.

기술적으로는 Istio control plane 설치에 선행하는 `base` Helm chart를 배포한다.

- 공식 Istio `base` chart `1.30.3`을 설치한다.
- Istio CRD, ClusterRole, ClusterRoleBinding과 webhook용 공통 리소스를 만든다.
- 대상 namespace는 `istio-system`이며 없으면 Argo CD가 생성한다.
- 대형 CRD의 client-side apply annotation 한도를 피하기 위해 server-side apply를 사용한다.

### 3. `clusters/dev/applications/istiod.yaml`

쉽게 설명하면, Istio 도로 전체를 관리하는 **교통 관제실**이다.

기술적으로는 Istio control plane인 `istiod`를 배포한다. 실제 application 요청을 직접 전달하는 data plane이 아니라 Envoy 설정, service discovery, 인증서 발급을 담당한다.

- 공식 `istiod` chart `1.30.3`을 설치한다.
- xDS API를 통해 Gateway와 sidecar Envoy에 listener, route, cluster 설정을 전달한다.
- workload ServiceAccount 신원을 바탕으로 mTLS 인증서를 발급하고 갱신한다.
- sidecar injection을 처리하는 mutating webhook을 제공한다.
- webhook field manager 충돌을 피하고 준비 후 검증 실패를 허용하지 않도록 `base.validationFailurePolicy: Fail`을 사용한다.

### 4. `clusters/dev/applications/istio-observability.yaml`

쉽게 설명하면, 아래의 Istio 측정기 파일들을 Argo CD가 찾아 설치하게 하는 **측정기 설치 주문서**다.

기술적으로는 이 Git 저장소의 `clusters/dev/monitoring/istio` 디렉터리를 source로 사용하는 child Application이다.

- ServiceMonitor와 PodMonitor를 `monitoring` namespace에 배포한다.
- 별도 Prometheus를 설치하지 않고 기존 kube-prometheus-stack을 재사용한다.
- Prometheus Operator CRD가 일시적으로 조회되지 않아도 Application 생성을 막지 않도록 `SkipDryRunOnMissingResource=true`를 사용한다.

### 5. `clusters/dev/monitoring/istio/istiod-servicemonitor.yaml`

쉽게 설명하면, Prometheus에게 관제실의 상태를 15초마다 확인하라고 알려 주는 **관제실 체온계**다.

기술적으로는 Prometheus Operator가 `istiod` Service의 metric endpoint를 scrape하도록 선언하는 ServiceMonitor다.

- `istio-system` namespace에서 `istio: pilot` label을 가진 Service를 선택한다.
- `http-monitoring` named port의 `/metrics`를 15초 간격, 10초 timeout으로 수집한다.
- `release: kube-prometheus-stack` label로 기존 Prometheus monitor 선택 규칙과 맞춘다.

### 6. `clusters/dev/monitoring/istio/envoy-podmonitor.yaml`

쉽게 설명하면, 모든 Envoy가 센 요청 수와 오류 수를 가져오는 **교통량 조사원**이다.

기술적으로는 모든 namespace에서 주입된 Envoy container의 Istio telemetry endpoint를 scrape하는 PodMonitor다.

- container 이름이 `istio-proxy`이고 `prometheus.io/scrape` annotation이 있는 target만 유지한다.
- Pod annotation의 Prometheus port와 Pod IP를 조합해 실제 scrape 주소로 바꾼다.
- 공식 Istio Prometheus Operator 예제의 relabeling을 사용해 IPv4와 IPv6 주소를 모두 처리한다.
- `namespace`와 `pod` target label을 추가해 Kiali가 metric의 workload 위치를 식별하게 한다.

### 7. `clusters/dev/applications/kiali.yaml`

쉽게 설명하면, Prometheus가 모은 숫자를 사람이 보기 쉬운 그림으로 바꾸는 **교통 지도 화면**이다.

기술적으로는 Kiali server Helm chart를 직접 설치하고 기존 observability backend와 연결한다.

- Kiali server chart `2.29.0`을 설치한다.
- `external_services`에 기존 Prometheus, Grafana, Tempo의 cluster 내부 주소를 지정한다.
- 인증 방식은 학습용 `anonymous`지만 Service type은 `ClusterIP`이므로 cluster 외부에서 직접 접근할 수 없다.
- Kiali 자체는 metric이나 trace 저장소가 아니며 Prometheus와 Tempo를 조회하는 UI·분석 계층이다.
- Grafana와 Tempo 연결이 실패해도 1차의 Kiali traffic graph 확인에는 Prometheus만 정상 연결되면 된다.

### 8. `clusters/dev/applications/argo-task-api-istio.yaml`

쉽게 설명하면, 기존 Task API와 비교하기 위한 **같은 가게의 두 번째 지점**이다. 판매하는 물건은 같지만 새 지점 앞뒤에는 교통을 기록하고 암호화하는 Envoy가 붙는다.

기술적으로는 기존 Task API Helm chart를 별도 release와 namespace에 배포하는 Argo CD Application이다.

- 기존 앱과 같은 image `6d8ead6`, RDS, Redis, Slack secret 설정을 사용한다.
- 비교 환경의 비용과 변수를 줄이기 위해 `replicaCount: 1`, `autoscaling.enabled: false`를 사용한다.
- application container request는 CPU `100m`, memory `128Mi`로 설정한다. Envoy request는 Istio의 기본 proxy resource 설정을 따른다.
- `managedNamespaceMetadata`로 `argo-task-api-istio` namespace에 `istio-injection=enabled` label을 선언한다.
- mutating webhook이 이 label을 감지해 Pod 생성 시 Envoy sidecar 구성을 자동 주입한다.
- 기존 dashboard와 같은 Grafana UID가 충돌하지 않도록 비교용 chart의 dashboard 생성을 끈다.

### 9. `clusters/dev/secretops/externalsecret-task-api-istio.yaml`

쉽게 설명하면, 비교용 앱이 필요한 비밀번호를 AWS 금고에서 가져오는 **안전한 열쇠 배달부**다.

기술적으로는 기존 ClusterSecretStore를 통해 AWS Secrets Manager 값을 별도 Kubernetes Secret으로 동기화하는 ExternalSecret이다.

- 새 AWS Secret을 만들지 않고 기존 remote key와 property를 재사용한다.
- DB 비밀번호, Redis 비밀번호, Slack webhook을 `task-api-runtime-secrets` Secret의 key로 매핑한다.
- Secret은 `argo-task-api-istio` namespace에 별도로 생성되므로 기존 앱의 Secret과 이름이 같아도 충돌하지 않는다.
- Git에는 secret 위치와 key 이름만 기록되고 실제 값은 저장되지 않는다.

### 10. `clusters/dev/applications/istio-edge.yaml`

쉽게 설명하면, Gateway와 HTTPRoute 파일을 Argo CD가 설치하게 하는 **새 출입구 공사 주문서**다.

기술적으로는 이 저장소의 `clusters/dev/istio/edge` 디렉터리를 source로 사용하는 child Application이다.

- destination을 `istio-ingress`로 지정하고 `CreateNamespace=true`로 Gateway namespace를 생성한다.
- Gateway와 HTTPRoute custom resource를 server-side apply로 관리한다.
- Gateway API CRD가 아직 discovery되지 않은 최초 설치 시점을 위해 `SkipDryRunOnMissingResource=true`를 사용한다.

### 11. `clusters/dev/istio/edge/gateway.yaml`

쉽게 설명하면, 인터넷 요청을 처음 받는 **새 정문**이다.

기술적으로는 `istio` GatewayClass를 사용하는 Gateway API `Gateway` resource다. Istio의 managed Gateway controller가 이 resource를 보고 실제 data plane workload를 생성한다.

- `istio-ingress` namespace에 `task-api-istio-gateway`라는 Gateway를 만든다.
- `http` listener는 `protocol: HTTP`, `port: 80`만 사용하며 TLS 설정은 없다.
- Istio가 Gateway용 Envoy Deployment, ServiceAccount, RBAC, LoadBalancer Service를 자동 생성·관리한다.
- `allowedRoutes.namespaces.from: All`로 다른 namespace의 HTTPRoute attachment를 허용한다. 실제 외부 노출 경로는 연결된 HTTPRoute가 제한한다.

### 12. `clusters/dev/istio/edge/task-api-httproute.yaml`

쉽게 설명하면, 정문에 도착한 요청이 어디로 갈지 알려 주는 **방향 표지판**이다.

기술적으로는 `argo-task-api-istio` namespace의 Gateway API `HTTPRoute` resource다.

- cross-namespace `parentRefs`로 `istio-ingress/task-api-istio-gateway`의 `http` listener에 attach한다.
- `Exact` path match를 사용해 `/version`, `/healthz`, `/readyz` 세 경로만 route한다.
- backendRef는 같은 namespace의 `argo-task-api-istio` Service 8000번 포트다.
- `hostnames`를 생략해 1차에서는 별도 DNS 없이 LoadBalancer hostname으로 요청할 수 있다.

## 커밋과 최초 적용 순서

현재 Argo CD root Application은 `deploy/dev`만 본다. 따라서 이 feature branch에 커밋이 있어도 아직 클러스터에는 적용되지 않는다.

최초 설치에서는 다음 커밋을 순서대로 `deploy/dev`에 반영하고, 매번 Argo CD가 `Synced / Healthy`가 된 뒤 다음 단계로 간다.

1. `9e4d10f` — Gateway API CRD
2. `5b8fca9` — Istio base와 istiod
3. `a956271` — Istio metric 수집과 Kiali
4. `e28f9fa` — 비교용 Task API와 Istio 외부 경로

처음부터 네 커밋을 squash해서 한 번에 배포하지 않는다. App of Apps의 sync wave는 child chart 설치가 완전히 끝날 때까지 기다리는 장치가 아니기 때문이다.

## 단계별 확인 명령

### Gateway API CRD 확인

```bash
kubectl get crd \
  gateways.gateway.networking.k8s.io \
  httproutes.gateway.networking.k8s.io \
  gatewayclasses.gateway.networking.k8s.io
```

### Istio control plane 확인

```bash
kubectl get pods,svc -n istio-system
kubectl get gatewayclass istio
```

`istiod` Pod와 Service가 준비되고 `istio` GatewayClass가 보여야 한다.

### Prometheus monitor와 Kiali 확인

```bash
kubectl get servicemonitor istiod -n monitoring
kubectl get podmonitor envoy-stats-monitor -n monitoring
kubectl get pod,svc -n istio-system -l app.kubernetes.io/name=kiali
kubectl port-forward -n istio-system svc/kiali 20001:20001
```

브라우저에서 `http://localhost:20001`로 접속한다.

### 비교용 앱과 sidecar 확인

```bash
kubectl get namespace argo-task-api-istio --show-labels
kubectl get externalsecret,secret -n argo-task-api-istio
kubectl get pods -n argo-task-api-istio
```

namespace 라벨에 `istio-injection=enabled`가 있고 Task API Pod가 `2/2 Ready`면 앱과 Envoy가 함께 준비된 것이다.

### Gateway 주소와 외부 요청 확인

```bash
kubectl get gateway task-api-istio-gateway -n istio-ingress -o wide
kubectl get httproute task-api-istio -n argo-task-api-istio
```

Gateway의 `ADDRESS`가 생기면 다음처럼 호출한다.

```bash
curl http://<GATEWAY_ADDRESS>/version
curl http://<GATEWAY_ADDRESS>/healthz
curl http://<GATEWAY_ADDRESS>/readyz
```

Kiali에서 `istio-ingress`와 `argo-task-api-istio` namespace를 선택하고 최근 5분 동안 요청을 반복하면 Gateway에서 Task API로 이어지는 선, 요청량, 오류율, 지연시간, mTLS 표시를 볼 수 있다.

## mTLS를 여기서는 어떻게 이해하면 되나

쉽게 설명하면, 외부 손님은 1차에서 암호화되지 않은 정문으로 들어오지만 정문 안쪽의 두 경비원은 서로 신분증을 확인하고 암호화해서 대화한다. Task API는 이 과정을 직접 처리하지 않고 앞에 붙은 Envoy에게 맡긴다.

기술적으로는 다음과 같다.

- 인터넷 client에서 Gateway LoadBalancer까지는 1차에서 TLS가 없는 **일반 HTTP**다.
- Istio Gateway Envoy에서 Task API sidecar Envoy까지는 두 endpoint가 mesh에 참여하므로 Istio의 **automatic mTLS**가 적용될 수 있다.
- 인증서 발급·갱신과 TLS handshake는 `istiod`와 Envoy가 처리하며 application code는 인증서 파일이나 TLS 설정을 알 필요가 없다.
- 1차에는 namespace 또는 workload 수준의 `PeerAuthentication STRICT`를 선언하지 않는다. 따라서 automatic mTLS는 사용되지만 평문 요청을 정책으로 강제 차단하는 단계는 아니다.
- 2차에서 `STRICT` 정책과 내부 호출 client를 추가해 mTLS 허용과 plaintext 거부를 각각 검증한다.

## 문제가 생기면 보는 순서

1. Argo CD Application이 `Synced / Healthy`인지 본다.
2. Gateway API CRD와 `istio` GatewayClass가 있는지 본다.
3. `istiod`, Kiali, 비교용 Task API Pod가 Ready인지 본다.
4. 비교용 ExternalSecret이 실제 Secret을 만들었는지 본다.
5. Gateway와 HTTPRoute의 `status.conditions`가 `Accepted=True`, `Programmed=True`인지 본다.
6. Prometheus에 `istio_requests_total` metric이 들어오는지 본다.
7. 요청을 만든 뒤 Kiali 시간 범위를 최근 5분으로 맞춘다.

## 되돌릴 때

설치의 반대 순서로 비교 경로 → Kiali와 monitor → istiod와 base → Gateway API CRD를 제거한다. 기존 Traefik과 기존 Task API는 별도 경로라 그대로 남는다.

## 용어 사전

### GitOps

Git에 기록된 설정을 시스템의 원하는 상태로 보고, controller가 실제 cluster 상태를 그 설정과 계속 일치시키는 운영 방식이다.

### Argo CD

Git 저장소의 Kubernetes manifest나 Helm chart를 읽어 cluster에 동기화하고, 차이가 생기면 감지하거나 자동 복구하는 GitOps controller다.

### Application과 App of Apps

Argo CD `Application`은 하나의 배포 source와 destination을 정의하는 custom resource다. App of Apps는 root Application이 여러 child Application manifest를 관리하는 구조다.

### Kubernetes manifest

Kubernetes resource의 원하는 상태를 YAML 또는 JSON으로 선언한 파일이다.

### Helm chart와 values

Helm chart는 여러 Kubernetes manifest를 재사용 가능한 template 묶음으로 만든 패키지다. values는 image tag, replica 수처럼 template에 넣을 설정값이다.

### CRD와 custom resource

CRD(CustomResourceDefinition)는 Kubernetes API에 새로운 resource 종류를 등록하는 schema다. CRD가 등록된 뒤 그 종류로 작성한 실제 객체를 custom resource라고 한다.

### Gateway API

Kubernetes에서 외부·내부 traffic routing을 표현하기 위한 표준 API다. 이 작업에서는 `GatewayClass`, `Gateway`, `HTTPRoute`를 사용한다.

### GatewayClass

Gateway를 어느 controller가 구현할지 지정한다. `gatewayClassName: istio`는 이 Gateway를 Istio가 관리한다는 뜻이다.

### Gateway

traffic을 받을 listener의 protocol, port, route 허용 범위를 선언한다. 선언 자체가 proxy process는 아니며 Istio controller가 이를 바탕으로 실제 Envoy workload와 Service를 만든다.

### HTTPRoute

HTTP 요청의 hostname, path, header 같은 조건과 요청을 보낼 backend를 선언한다. 이 작업에서는 세 개의 exact path를 Task API Service로 연결한다.

### Listener

Gateway가 traffic을 받을 protocol과 port의 조합이다. 현재 listener는 HTTP 80번 하나다.

### Istio와 service mesh

Istio는 application code 밖의 proxy를 이용해 service 간 routing, 보안, 관측을 제공하는 service mesh다. service mesh는 여러 service 사이의 통신을 공통 계층에서 관리하는 구조를 뜻한다.

### Control plane과 data plane

Control plane은 proxy에 설정과 인증서를 전달하는 관리 영역이며 이 구성에서는 `istiod`가 담당한다. Data plane은 실제 요청을 전달하는 영역이며 Gateway와 sidecar의 Envoy가 담당한다.

### istiod

Istio control plane process다. service와 route를 발견해 Envoy 설정을 만들고, workload 신원을 확인해 mTLS 인증서를 발급한다.

### Envoy

Istio가 data plane에 사용하는 L4/L7 proxy다. 요청을 전달하면서 metric을 만들고 TLS, load balancing, retry 같은 network 기능을 수행할 수 있다.

### Sidecar와 sidecar injection

Sidecar는 주 application container와 같은 Pod에 붙어 보조 기능을 수행하는 container다. Sidecar injection은 Pod 생성 시 mutating webhook이 Envoy 구성을 자동으로 추가하는 과정이다.

### Pod와 container

Pod는 Kubernetes가 배치하는 최소 실행 단위다. 하나 이상의 container가 network와 일부 storage를 공유한다. 비교용 Task API Pod에는 application container와 Envoy가 함께 실행된다.

### Deployment와 replica

Deployment는 원하는 Pod template과 개수를 관리하는 controller resource다. Replica는 동시에 유지할 Pod 복제본 수다.

### Namespace

Kubernetes resource를 논리적으로 구분하는 경계다. 이름이 같은 Secret이나 Service도 namespace가 다르면 서로 다른 resource다.

### Label과 selector

Label은 resource에 붙이는 key-value 식별자다. Selector는 label 조건으로 관리하거나 관측할 resource를 선택한다.

### Service

변할 수 있는 여러 Pod IP 앞에 안정적인 DNS 이름과 virtual IP를 제공하고, selector에 맞는 Pod로 traffic을 분배하는 Kubernetes resource다.

### ClusterIP와 LoadBalancer

`ClusterIP` Service는 기본적으로 cluster 내부에서만 접근한다. `LoadBalancer` Service는 cloud provider와 연동해 cluster 외부에서 접근할 load balancer 주소를 만든다.

### Managed Gateway

사용자가 Gateway resource만 선언하면 Istio가 Gateway용 Deployment, Service, ServiceAccount, RBAC 등을 자동 생성하고 관리하는 방식이다.

### Prometheus와 metric

Prometheus는 HTTP endpoint를 주기적으로 scrape해 시계열 metric을 저장하고 조회하는 시스템이다. Metric은 요청 수, 오류 수, 처리 시간처럼 시간에 따라 변하는 수치 데이터다.

### Prometheus Operator, ServiceMonitor, PodMonitor

Prometheus Operator는 Kubernetes custom resource를 보고 Prometheus scrape 설정을 생성한다. ServiceMonitor는 Service를 기준으로, PodMonitor는 Pod를 기준으로 scrape target을 선택한다.

### Telemetry

시스템 동작을 관찰하기 위해 수집하는 metric, log, trace 데이터를 통칭한다. 1차 Kiali graph는 주로 Istio metric을 사용한다.

### Kiali

Istio 설정과 Prometheus metric을 조회해 mesh topology, traffic, 오류율, 지연시간, 보안 상태를 보여 주는 관리·관측 UI다. 장기 metric 저장소는 아니다.

### Grafana

Prometheus 같은 data source를 조회해 dashboard로 시각화하는 도구다. Kiali와 역할이 겹치는 부분이 있지만 Grafana는 범용 dashboard에, Kiali는 Istio mesh 분석에 초점이 있다.

### Tempo와 trace

Trace는 하나의 요청이 여러 component를 통과한 과정을 span들의 연결로 기록한 데이터다. Tempo는 trace를 저장하고 조회하는 backend다.

### TLS와 mTLS

TLS는 client가 server 인증서를 확인하고 통신을 암호화하는 protocol이다. mTLS(mutual TLS)는 server뿐 아니라 client도 인증서를 제시해 양쪽이 서로의 신원을 확인한다.

### Automatic mTLS와 STRICT

Automatic mTLS는 Istio가 source와 destination의 mesh 참여 여부를 감지해 Envoy 사이에 mTLS를 자동 선택하는 기능이다. `PeerAuthentication`의 `STRICT` mode는 destination workload가 plaintext 연결을 받지 않도록 정책으로 강제한다.

### HPA와 resource request

HPA(HorizontalPodAutoscaler)는 CPU 같은 지표에 따라 Pod replica 수를 자동 조절한다. Resource request는 scheduler가 Pod 배치에 사용하는 최소 CPU·memory 예약량이다.

### ExternalSecret과 AWS Secrets Manager

AWS Secrets Manager는 실제 secret 값을 AWS에 저장한다. ExternalSecret은 외부 secret 저장소의 값을 읽어 Kubernetes Secret으로 동기화하라는 선언이다.

### ServiceAccount

Pod가 Kubernetes와 mesh 안에서 사용하는 workload 신원이다. Istio는 이 신원을 바탕으로 workload 인증서를 발급한다.

### Webhook

Kubernetes API 요청이 저장되기 전 resource를 수정하거나 검증하는 HTTP callback이다. Istio의 mutating webhook은 Pod에 sidecar 구성을 주입한다.

### Server-side apply

적용할 resource의 field 소유권과 병합을 Kubernetes API server가 관리하는 방식이다. 큰 CRD의 client-side annotation 크기 문제와 여러 controller의 field 관리 충돌을 줄인다.

### Dry-run과 `SkipDryRunOnMissingResource`

Dry-run은 실제 저장 전에 manifest가 적용 가능한지 검사하는 과정이다. CRD가 아직 discovery되지 않은 최초 설치에서는 custom resource 검사에 실패할 수 있어 해당 Argo CD 옵션으로 그 검사를 일시적으로 건너뛴다.

### Sync wave

Argo CD가 한 번의 sync에서 resource 적용 순서를 나누는 숫자다. 작은 숫자가 먼저 적용된다. App of Apps에서는 child Application 객체의 생성 순서를 정할 뿐, 각 child가 배포하는 chart의 완료까지 완전히 보장하지는 않는다.

### Port-forward

로컬 컴퓨터의 port를 cluster 내부 Pod나 Service port에 임시 연결하는 `kubectl` 기능이다. Kiali를 외부에 공개하지 않고 로컬 브라우저로 확인할 때 사용한다.
