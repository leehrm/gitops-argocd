# Istio Gateway·Kiali 외부 트래픽 구현 및 변경 설명서

기존 Traefik 경로를 유지하면서 동일한 Task API를 별도 Istio 경로로 배포하고, 외부 요청이 Istio Gateway와 Envoy sidecar를 거쳐 애플리케이션에 도달하는 흐름을 Kiali에서 관찰할 수 있도록 구현한 결과를 정리함.

## 구현 목표

- 기존 Traefik 기반 Task API를 비교 기준선으로 유지함
- 비교용 Task API를 별도 namespace와 workload로 배포함
- Istio Gateway를 외부 진입점으로 구성함
- Cloudflare DNS와 Let's Encrypt HTTPS 인증서를 자동 관리함
- Gateway에서 Task API까지의 traffic을 mTLS로 전달함
- Envoy metric을 기존 Prometheus로 수집하고 Kiali에서 시각화함
- 외부에는 조회·상태 확인용 endpoint만 노출함

## 구현 결과

| 기능 | 구현 결과 | 주요 리소스 |
|---|---|---|
| Gateway API 지원 | 표준 Gateway API CRD `v1.5.1` 설치 | `gateway-api-crds.yaml` |
| Istio control plane | Istio `1.30.3`의 CRD와 `istiod` 설치 | `istio-base.yaml`, `istiod.yaml` |
| 비교용 workload | 별도 namespace에 Task API와 Envoy sidecar 배포 | `argo-task-api-istio.yaml` |
| 외부 진입점 | Istio managed Gateway와 AWS LoadBalancer 생성 | `gateway.yaml` |
| DNS 자동화 | HTTPRoute hostname을 감시해 Cloudflare record 관리 | `external-dns.yaml` |
| HTTPS | 두 public hostname의 Let's Encrypt 인증서 발급 및 갱신 | `clusterissuers.yaml`, `istio-gateway-certificate.yaml` |
| Route 제한 | Task API의 세 GET endpoint만 외부 연결 | `task-api-httproute.yaml` |
| HTTP redirect | 80번 port 요청을 HTTPS로 `301` redirect | `https-redirect-httproute.yaml` |
| Telemetry 수집 | `istiod`와 Envoy metric을 기존 Prometheus로 수집 | `istiod-servicemonitor.yaml`, `envoy-podmonitor.yaml` |
| Kiali | Gateway와 workload traffic, 오류율, latency, mTLS 상태 제공 | `kiali.yaml`, `kiali-httproute.yaml` |
| Workload 보안 | Task API에 `STRICT` mTLS와 Gateway identity 기반 접근 제한 적용 | `argo-task-api-istio-peerauthentication.yaml`, `argo-task-api-istio-authorizationpolicy.yaml` |

## 기존 비교 기준선

비교 기준선은 새 구조와 동작 차이를 확인하기 위해 변경하지 않고 남겨 둔 기존 환경임. 같은 Pod에 진입점만 하나 추가한 구조가 아님. 기존 환경과 Istio 환경은 애플리케이션 image와 DB·Redis를 공유하지만 namespace, Deployment, Pod, Service, 외부 hostname과 ingress 경로가 서로 다른 별도 배포임.

| 항목 | 기존 비교 기준선 | Istio 비교 환경 |
|---|---|---|
| public hostname | `task-api.lhrm-lab.com` | `task-api-istio.lhrm-lab.com` |
| namespace | `argo-task-api` | `argo-task-api-istio` |
| 외부 진입점 | Traefik | Istio Gateway Envoy |
| Task API Pod | 기존 별도 Pod, Envoy sidecar 없음 | 비교용 별도 Pod, Envoy sidecar 있음 |
| public TLS 종료 | Traefik | Istio Gateway Envoy |
| proxy 이후 통신 | Traefik → Task API: cluster HTTP | Gateway → Task API sidecar: 별도 mTLS |
| Kiali traffic graph | mesh 외부이므로 표시하지 않음 | Gateway → Task API edge 표시 |

### 기존 Traefik 환경을 유지한 이유

이번 구현의 목적은 기존 ingress를 즉시 교체하는 것이 아니라 Traefik과 Istio의 외부 traffic 처리 방식을 비교하는 것임. 기존 경로까지 동시에 변경하면 Gateway, sidecar, mTLS 중 무엇이 latency, 오류 또는 routing 차이를 만들었는지 구분하기 어려워짐.

- **비교 가능성**: 같은 애플리케이션의 Traefik 경로와 Istio 경로를 동시에 호출해 결과와 metric을 비교할 수 있음
- **기존 경로 보호**: 기존 DNS, 인증서, Traefik routing과 호출 방식에 영향을 주지 않음
- **장애 범위 분리**: Istio Gateway 또는 sidecar에 문제가 생겨도 기존 Task API 경로는 유지됨
- **즉시 rollback**: Istio 비교 경로만 제거하면 되며 Traefik 복구 작업이 필요 없음
- **TLS 비교**: public TLS가 끝나는 지점과 proxy 이후 암호화 방식의 차이를 직접 확인할 수 있음

Traefik을 반드시 유지해야 하는 기술적 제약이 있는 것은 아님. 비교와 검증이 끝난 뒤 실제 ingress를 Istio로 교체할지는 별도의 전환 작업으로 판단함.

## 외부 traffic과 TLS 종료 지점

`TLS termination(TLS 종료)`은 하나의 TLS 연결을 복호화하고 끝내는 지점을 뜻함. TLS가 종료된 뒤 다른 인증서와 목적을 사용하는 새 TLS 또는 mTLS 연결을 시작할 수 있음.

### 기존 Traefik 경로

기존 hostname의 public TLS는 Traefik에서 종료됨. Traefik이 요청을 복호화한 뒤 기존 Task API Service와 Pod에는 cluster HTTP로 전달함.

```text
사용자
  == public HTTPS ==>
[Traefik: public TLS 종료]
  -- cluster HTTP -->
[argo-task-api Service]
  -- HTTP -->
[기존 Task API Pod: Envoy sidecar 없음]
```

### Istio Task API 경로

새 hostname의 public TLS는 Istio Gateway Envoy에서 종료됨. Gateway는 Task API sidecar와 별도의 Istio mTLS 연결을 시작함. Task API sidecar가 mesh mTLS를 종료한 뒤 같은 Pod의 애플리케이션 container에는 local HTTP로 전달함.

```text
사용자
  == public HTTPS / Let's Encrypt ==>
[Istio Gateway Envoy: public TLS 종료]
  == 별도 Istio mTLS / Istio workload 인증서 ==>
[Task API Envoy sidecar: mesh mTLS 종료]
  -- Pod 내부 local HTTP -->
[비교용 Task API container:8000]
```

사용자부터 애플리케이션까지 하나의 TLS 연결이 그대로 이어지는 구조가 아님. Gateway에서 public TLS가 끝나고 Gateway와 sidecar 사이에서 mesh mTLS 연결이 새로 시작됨. HTTP 80번 요청은 동일 Gateway에서 HTTPS로 `301` redirect함.

### Kiali 경로

`kiali.lhrm-lab.com`의 public TLS도 Istio Gateway에서 종료됨. 현재 Kiali workload에는 Envoy sidecar를 주입하지 않으므로 Gateway에서 Kiali ClusterIP Service 20001번 port까지는 cluster HTTP로 전달함. Kiali는 `auth.strategy: token`을 사용하므로 HTTPS 연결 후 Kubernetes ServiceAccount token으로 로그인해야 함.

```text
사용자
  == public HTTPS ==>
[Istio Gateway Envoy: public TLS 종료]
  -- cluster HTTP -->
[Kiali Service:20001]
  -->
[Kiali]
```

## 기능별 구현 내용

### Gateway API와 Istio control plane

#### `clusters/dev/applications/gateway-api-crds.yaml`

- 공식 Gateway API standard CRD `v1.5.1` 설치함
- `GatewayClass`, `Gateway`, `HTTPRoute` API를 cluster에 등록함
- 대형 CRD를 안정적으로 관리하기 위해 server-side apply를 사용함
- 다른 Istio Application보다 먼저 생성되도록 sync wave `-5`를 적용함

#### `clusters/dev/applications/istio-base.yaml`

- 공식 Istio `base` chart `1.30.3` 설치함
- Istio CRD, cluster RBAC, validation webhook 공통 리소스를 구성함
- controller가 관리하는 validation webhook 차이를 Argo CD drift로 오인하지 않도록 예외를 설정함

#### `clusters/dev/applications/istiod.yaml`

- Istio control plane인 `istiod` `1.30.3` 설치함
- Gateway와 sidecar Envoy에 listener, route, cluster 설정을 전달함
- workload ServiceAccount 신원을 기반으로 mTLS 인증서를 발급하고 갱신함
- namespace label을 감지해 Envoy sidecar를 주입하는 webhook을 제공함

### 비교용 Task API와 sidecar

#### `clusters/dev/applications/argo-task-api-istio.yaml`

- 기존 Helm chart를 `argo-task-api-istio` namespace의 별도 Application으로 배포함
- `managedNamespaceMetadata`에 `istio-injection=enabled`를 선언함
- Pod 생성 시 application container 옆에 `istio-proxy` Envoy container를 자동 주입함
- 기존 환경과 독립된 Deployment, Service, Pod를 사용함
- 비교 환경의 비용과 변수를 줄이기 위해 replica 1개와 HPA 비활성화를 적용함
- 외부 ingress는 Helm chart에서 비활성화하고 Gateway API HTTPRoute만 사용함

#### `clusters/dev/secretops/externalsecret-task-api-istio.yaml`

- 비교용 Task API의 DB와 Redis credential을 AWS Secrets Manager에서 별도 Kubernetes Secret으로 동기화함
- Git에는 remote key와 property 이름만 기록하고 실제 secret 값은 저장하지 않음

### Istio Gateway와 HTTPRoute

#### `clusters/dev/applications/istio-edge.yaml`

- `clusters/dev/istio/edge`의 Gateway, HTTPRoute, Certificate를 관리함
- `istio-ingress` namespace를 생성하고 server-side apply를 사용함

#### `clusters/dev/istio/edge/gateway.yaml`

- `gatewayClassName: istio`인 managed Gateway를 선언함
- Istio가 Gateway용 Envoy Deployment, ServiceAccount, RBAC와 AWS LoadBalancer Service를 생성함
- HTTP 80 listener와 hostname별 HTTPS 443 listener를 구성함
- `task-api-istio.lhrm-lab.com`, `kiali.lhrm-lab.com` listener가 동일 TLS Secret을 참조함

#### `clusters/dev/istio/edge/task-api-httproute.yaml`

- `task-api-istio.lhrm-lab.com` 요청을 비교용 Task API Service 8000번 port로 전달함
- 세 endpoint 경로만 backend로 연결하고 HTTP method는 AuthorizationPolicy에서 `GET`으로 제한함
- `/tasks` 같은 쓰기 endpoint는 public route에 포함하지 않음

#### `clusters/dev/istio/edge/kiali-httproute.yaml`

- `kiali.lhrm-lab.com` 요청을 Kiali Service 20001번 port로 전달함
- Kiali UI 전체 경로를 위해 `/` prefix match를 사용함

#### `clusters/dev/istio/edge/https-redirect-httproute.yaml`

- 두 hostname의 HTTP 요청을 HTTPS로 `301` redirect함
- cert-manager HTTP-01 solver가 사용하는 HTTP listener를 유지함

### DNS와 HTTPS 자동화

#### `clusters/dev/applications/external-dns.yaml`

- ExternalDNS `1.21.1`이 기존 Ingress와 Gateway API HTTPRoute를 함께 감시함
- HTTPRoute의 hostname과 Gateway LoadBalancer 주소를 이용해 Cloudflare DNS record를 생성·갱신함
- `lhrm-lab.com` domain만 관리하고 `upsert-only` 정책으로 자동 삭제를 제한함
- TXT registry와 owner ID를 사용해 다른 ExternalDNS 인스턴스와 record 소유권 충돌을 방지함

Cloudflare에 A/CNAME record를 수동으로 등록하는 구조가 아님. Cloudflare zone과 API token을 준비하면 ExternalDNS가 HTTPRoute hostname을 기준으로 record를 관리함.

#### `clusters/dev/cert-manager/clusterissuers.yaml`

- 기존 Traefik용 ACME issuer와 별도로 Gateway API HTTPRoute solver를 사용하는 issuer를 구성함
- `letsencrypt-production-gateway`가 Gateway의 HTTP listener를 통해 HTTP-01 challenge를 처리함
- 기존 Traefik 인증서 발급 경로는 변경하지 않음

#### `clusters/dev/istio/edge/istio-gateway-certificate.yaml`

- `task-api-istio.lhrm-lab.com`, `kiali.lhrm-lab.com`을 포함하는 Let's Encrypt 인증서를 요청함
- 발급된 인증서를 `istio-ingress/istio-gateway-lhrm-lab-com-tls` Secret에 저장함
- Gateway의 두 HTTPS listener가 동일 Secret을 참조함

### Prometheus와 Kiali

#### `clusters/dev/applications/istio-observability.yaml`

- 별도 Prometheus를 추가하지 않고 기존 kube-prometheus-stack을 재사용함
- Istio ServiceMonitor와 PodMonitor를 `monitoring` namespace에 배포함

#### `clusters/dev/monitoring/istio/istiod-servicemonitor.yaml`

- `istiod`의 `/metrics` endpoint를 15초 간격으로 수집함
- `release: kube-prometheus-stack` label로 기존 Prometheus 선택 규칙과 연결함

#### `clusters/dev/monitoring/istio/envoy-podmonitor.yaml`

- 모든 namespace의 `istio-proxy` container telemetry endpoint를 수집함
- Pod annotation의 port와 IP를 조합해 IPv4·IPv6 scrape 주소를 구성함
- `namespace`, `pod` label을 추가해 Kiali가 metric의 workload 위치를 식별할 수 있게 함

#### `clusters/dev/applications/kiali.yaml`

- Kiali server chart `2.29.0` 설치함
- 기존 Prometheus, Grafana, Tempo의 cluster 내부 주소와 연결함
- Kiali가 Prometheus metric을 조회해 topology, 요청량, 오류율, latency와 mTLS 상태를 표시함
- public 접근에는 Kubernetes token 인증을 요구함
- Kiali는 metric과 trace를 직접 저장하지 않는 조회·분석 계층임

### Workload 접근 정책

#### `clusters/dev/istio/policies/argo-task-api-istio-peerauthentication.yaml`

- 비교용 Task API namespace에 `PeerAuthentication STRICT`를 적용함
- sidecar 없이 전달되는 plaintext 연결을 거부함

#### `clusters/dev/istio/policies/argo-task-api-istio-authorizationpolicy.yaml`

- Istio Gateway ServiceAccount principal만 Task API workload를 호출할 수 있게 제한함
- 8000번 port의 `GET /version`, `/healthz`, `/readyz`만 허용함
- mTLS 암호화와 별도로 호출 주체와 허용 endpoint를 검증함

## Kiali에서 확인 가능한 범위

확인 가능함:

- `istio-ingress` Gateway에서 `argo-task-api-istio` workload로 이어지는 edge
- 요청량, 성공률, 오류율, latency
- Task API Pod와 Envoy sidecar 상태
- destination traffic의 mTLS 적용 상태
- Istio config와 workload health

확인할 수 없음:

- 인터넷 client와 AWS LoadBalancer를 각각 독립 graph node로 표시하는 기능
- Cloudflare DNS 조회 과정
- browser와 Gateway 사이 TLS handshake의 세부 packet
- 애플리케이션 business logic 내부 함수 호출

Kiali graph는 Envoy가 생성하고 Prometheus가 저장한 mesh telemetry를 기반으로 함. 따라서 graph의 시작점은 인터넷이 아니라 mesh에 참여하는 Gateway 또는 workload로 표시됨.

Task API에서 Notification Service로 이어지는 내부 service traffic은 [Notification Service 구현 설명서](./istio-kiali-notification-service-changes.md)에서 별도로 설명함.

## 확인 방법

### Kubernetes와 Argo CD 상태

```bash
argocd app get gateway-api-crds
argocd app get istio-base
argocd app get istiod
argocd app get istio-edge
argocd app get istio-observability
argocd app get kiali
argocd app get argo-task-api-istio

kubectl get gateway,httproute -A
kubectl get certificate -n istio-ingress
kubectl get pods -n istio-system
kubectl get pods -n istio-ingress
kubectl get pods -n argo-task-api-istio
```

정상 기준:

- Argo CD Application이 `Synced / Healthy`임
- Gateway condition이 `Accepted=True`, `Programmed=True`임
- Certificate가 `Ready=True`임
- 비교용 Task API Pod가 application과 Envoy를 포함해 `2/2 Running`임

### 외부 DNS와 HTTPS

```bash
dig +short task-api-istio.lhrm-lab.com
dig +short kiali.lhrm-lab.com

curl -I http://task-api-istio.lhrm-lab.com/version
curl -sS https://task-api-istio.lhrm-lab.com/version
curl -sS https://task-api-istio.lhrm-lab.com/healthz
curl -sS https://task-api-istio.lhrm-lab.com/readyz
```

정상 기준:

- 두 hostname이 Istio Gateway LoadBalancer 주소로 해석됨
- HTTP 요청이 HTTPS로 `301` redirect됨
- HTTPS 인증서의 hostname 검증이 성공함
- 세 Task API endpoint가 정상 응답함

### Kiali login과 traffic graph

```bash
kubectl -n istio-system create token kiali
```

1. `https://kiali.lhrm-lab.com` 접속
2. 생성한 token으로 로그인
3. `istio-ingress`, `argo-task-api-istio` namespace 선택
4. Task API의 `/version`, `/healthz`, `/readyz` 반복 호출
5. Gateway → Task API edge, 요청량, 오류율, latency와 mTLS 표시 확인

## 장애 확인 순서

1. ExternalDNS가 두 hostname record를 생성했는지 확인
2. Certificate와 ACME challenge 상태 확인
3. Gateway의 `Accepted`, `Programmed` condition 확인
4. HTTPRoute의 `Accepted`, `ResolvedRefs` condition 확인
5. 비교용 Task API Pod의 `2/2 Ready`와 sidecar injection 확인
6. `PeerAuthentication`과 `AuthorizationPolicy` 확인
7. Prometheus에 `istio_requests_total` metric이 수집되는지 확인
8. Kiali 시간 범위와 선택 namespace 확인

## Rollback 범위

외부 비교 경로만 제거하려면 `istio-edge`와 `argo-task-api-istio` 리소스를 Git에서 제거한 뒤 Argo CD sync와 prune을 수행함. 기존 Traefik과 `argo-task-api`는 별도 hostname과 리소스를 사용하므로 영향받지 않음.

ExternalDNS가 `upsert-only` 정책을 사용하므로 HTTPRoute 제거만으로 Cloudflare DNS record가 자동 삭제되지는 않음. 외부 경로를 완전히 정리할 때에는 두 Istio hostname의 DNS record도 별도로 확인함.

현재 Notification Service 내부 traffic도 Istio를 사용하므로 외부 경로 rollback을 이유로 `istiod`, `istio-base`, workload policy 전체를 함께 제거하면 안 됨. control plane과 CRD 제거는 mesh를 사용하는 workload가 더 이상 없을 때 별도 작업으로 수행해야 함.

## 의도적으로 제한한 범위

- 기존 Traefik ingress 교체
- public `/tasks` 쓰기 route
- Kiali OpenID/OAuth 로그인
- Cloudflare proxy mode
- internet client와 AWS LoadBalancer를 Kiali node로 표시하는 기능
- application trace context 자동 전파

## 용어

### Gateway API

Kubernetes에서 외부·내부 traffic routing을 표현하는 표준 API임. 이 구현에서는 `GatewayClass`, `Gateway`, `HTTPRoute`를 사용함.

### Istio control plane과 data plane

Control plane은 Envoy에 route, service discovery와 인증서를 전달하는 관리 영역이며 `istiod`가 담당함. Data plane은 실제 요청을 전달하는 Gateway와 sidecar Envoy임.

### Sidecar

애플리케이션 container와 같은 Pod에서 실행되는 보조 proxy임. Istio의 Envoy sidecar가 inbound·outbound traffic, mTLS와 telemetry를 처리함.

### Telemetry

시스템 상태를 관찰하기 위해 수집하는 metric, log, trace를 통칭함. Kiali traffic graph는 주로 Envoy가 생성한 Istio metric을 사용함.

### TLS와 mTLS

TLS는 일반적으로 client가 server 인증서를 확인하고 통신을 암호화함. mTLS는 client와 server가 모두 인증서를 제시해 양쪽 workload의 신원을 확인함.

### PeerAuthentication과 AuthorizationPolicy

`PeerAuthentication`은 workload가 mTLS 연결만 받을지 결정함. `AuthorizationPolicy`는 인증된 workload identity 중 누가 어떤 port와 endpoint를 호출할 수 있는지 제한함.

### Kiali

Istio 설정과 Prometheus metric을 조회해 mesh topology, traffic, 오류율, latency와 보안 상태를 제공하는 관리·관측 UI임.
