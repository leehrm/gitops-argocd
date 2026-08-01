# Istio·Kiali 1차 변경 설명서

이 문서는 이번 1차 작업에서 무엇을 추가했는지 12살도 이해할 수 있게 설명한다.

## 한 문장으로 설명

기존 Traefik 길은 그대로 두고, 똑같은 Task API 복사본으로 가는 **새 Istio 길**을 하나 만들었다. Kiali는 새 길에서 요청이 어디를 지나가는지 보여주는 교통 지도다.

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

## 이번에 일부러 하지 않은 것

- 기존 Traefik과 기존 `argo-task-api`는 수정하지 않았다.
- DNS와 HTTPS 인증서는 붙이지 않았다. 1차는 LoadBalancer 주소로 HTTP 테스트를 한다.
- Kiali를 인터넷에 공개하지 않았다. `kubectl port-forward`로만 접속한다.
- `/tasks` 같은 쓰기 API는 공개하지 않았다. `/version`, `/healthz`, `/readyz`만 허용한다.
- `PeerAuthentication STRICT` 정책과 내부 서비스 간 호출은 2차 작업으로 남겼다.

## 파일별 변경 설명

### 1. `clusters/dev/applications/gateway-api-crds.yaml`

Kubernetes가 `Gateway`와 `HTTPRoute`라는 새 단어를 알아듣게 하는 **사전**이다.

- 공식 Gateway API `v1.5.1`의 standard CRD를 설치한다.
- 큰 CRD도 안전하게 저장되도록 `ServerSideApply=true`를 사용한다.
- Istio보다 먼저 준비되도록 sync wave를 `-5`로 둔다.

### 2. `clusters/dev/applications/istio-base.yaml`

Istio가 사용할 표지판과 규칙을 먼저 깔아 주는 **도로 기초 공사**다.

- 공식 Istio `base` chart `1.30.3`을 설치한다.
- Istio CRD와 공통 권한을 만든다.
- 큰 CRD를 위해 server-side apply를 사용한다.

### 3. `clusters/dev/applications/istiod.yaml`

Istio 도로 전체를 관리하는 **교통 관제실**이다.

- 공식 `istiod` chart `1.30.3`을 설치한다.
- Envoy들에게 어디로 요청을 보낼지 알려 준다.
- sidecar 인증서를 발급하고 자동으로 갱신한다.
- webhook을 안전하게 관리하도록 `base.validationFailurePolicy: Fail`을 사용한다.

### 4. `clusters/dev/applications/istio-observability.yaml`

아래의 Istio 측정기 파일들을 Argo CD가 찾아 설치하게 하는 **측정기 설치 주문서**다.

- `clusters/dev/monitoring/istio` 경로를 감시한다.
- 기존 `monitoring` namespace와 Prometheus를 재사용한다.
- Prometheus CRD가 준비되는 동안의 dry-run 오류를 피하도록 설정했다.

### 5. `clusters/dev/monitoring/istio/istiod-servicemonitor.yaml`

Prometheus에게 관제실의 상태를 15초마다 확인하라고 알려 주는 **관제실 체온계**다.

- `istio-system`의 `istiod` Service를 찾는다.
- `http-monitoring` 포트의 `/metrics`를 수집한다.

### 6. `clusters/dev/monitoring/istio/envoy-podmonitor.yaml`

모든 Envoy가 센 요청 수와 오류 수를 가져오는 **교통량 조사원**이다.

- `istio-proxy` container만 골라서 수집한다.
- sidecar가 알려 준 Prometheus 포트로 접속한다.
- IPv4와 IPv6 Pod 주소를 모두 처리한다.
- namespace와 Pod 이름을 metric에 남겨 Kiali가 대상을 구분하게 한다.

### 7. `clusters/dev/applications/kiali.yaml`

Prometheus가 모은 숫자를 사람이 보기 쉬운 그림으로 바꾸는 **교통 지도 화면**이다.

- Kiali server chart `2.29.0`을 설치한다.
- 기존 Prometheus, Grafana, Tempo 주소를 사용한다.
- 학습용이라 로그인은 `anonymous`지만 Service는 `ClusterIP`라 외부에 공개되지 않는다.
- Kiali 자체는 metric이나 trace를 저장하지 않는다.

### 8. `clusters/dev/applications/argo-task-api-istio.yaml`

기존 Task API와 비교하기 위한 **같은 가게의 두 번째 지점**이다.

- 기존 앱과 같은 image `6d8ead6`, RDS, Redis, Slack secret 설정을 사용한다.
- 비교용이라 replica는 1개이고 HPA는 끈다.
- 앱의 CPU request만 `100m`, memory request는 `128Mi`로 낮춘다.
- `argo-task-api-istio` namespace에 `istio-injection=enabled` 라벨을 붙인다.
- 이 라벨 때문에 Task API Pod 옆에 Envoy sidecar가 자동으로 들어간다.
- 기존 dashboard와 같은 UID가 겹치지 않도록 비교용 Grafana dashboard 생성은 끈다.

### 9. `clusters/dev/secretops/externalsecret-task-api-istio.yaml`

비교용 앱이 필요한 비밀번호를 AWS 금고에서 가져오는 **안전한 열쇠 배달부**다.

- 새 AWS Secret을 만들지 않는다.
- 기존 앱과 같은 DB 비밀번호, Redis 비밀번호, Slack webhook을 읽는다.
- Kubernetes Secret은 비교 namespace 안에 따로 만든다.
- Git에는 실제 비밀번호가 들어가지 않는다.

### 10. `clusters/dev/applications/istio-edge.yaml`

Gateway와 HTTPRoute 파일을 Argo CD가 설치하게 하는 **새 출입구 공사 주문서**다.

- `clusters/dev/istio/edge` 경로를 감시한다.
- Gateway가 들어갈 `istio-ingress` namespace를 만든다.
- Gateway API CRD가 설치되는 첫 순간의 dry-run 오류를 피한다.

### 11. `clusters/dev/istio/edge/gateway.yaml`

인터넷 요청을 처음 받는 **새 정문**이다.

- Istio의 managed Gateway를 사용한다.
- HTTP 80번 포트만 연다.
- Istio가 Gateway용 Envoy Deployment와 LoadBalancer Service를 자동으로 만든다.
- 다른 namespace의 HTTPRoute가 이 문을 사용할 수 있게 허용한다.

### 12. `clusters/dev/istio/edge/task-api-httproute.yaml`

정문에 도착한 요청이 어디로 갈지 알려 주는 **방향 표지판**이다.

- `/version`, `/healthz`, `/readyz`와 정확히 일치하는 요청만 받는다.
- 요청을 `argo-task-api-istio` Service의 8000번 포트로 보낸다.
- hostname을 제한하지 않아 1차에서는 LoadBalancer 주소로 바로 테스트할 수 있다.

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

- 인터넷에서 Gateway까지는 1차에서 **일반 HTTP**다.
- Gateway의 Envoy에서 Task API sidecar Envoy까지는 Istio의 **automatic mTLS**를 사용할 수 있다.
- 앱 코드는 인증서를 직접 읽거나 TLS 코드를 작성하지 않는다. 두 Envoy가 대신 처리한다.
- 2차에서는 `STRICT` 정책과 내부 호출용 client를 추가해 평문 통신을 명시적으로 막는 실험을 한다.

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
