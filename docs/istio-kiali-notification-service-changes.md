# Istio·Kiali Notification Service 구현 설명서

이 문서는 기존 Traefik 환경을 유지하면서 비교용 Istio 환경에 실제 내부 서비스 통신을 추가한 결과를 설명한다. 전체 구조, 파일별 변경, 보안 정책과 검증 결과를 정리한다.

## 요약

기존 Task API는 완료 알림을 직접 Slack에 보낸다. 비교용 Istio Task API는 알림 내용을 별도 Notification Service에 전달하고, Notification Service가 Slack에 보낸다. 두 서비스 사이에는 Envoy sidecar가 있어 통신을 암호화하고 신원을 확인하며, Kiali는 그 흐름을 그래프로 보여 준다.

## “기존 비교 기준선”이란

비교 기준선은 새 구조와 비교하기 위해 **바꾸지 않고 남겨 둔 기존 환경**이다. 같은 Pod에 진입점만 하나 더 붙인 것이 아니다. 애플리케이션 image와 DB·Redis는 공유하지만, 기존 환경과 Istio 환경은 namespace, Deployment, Pod, Service, 외부 진입 경로가 서로 다른 별도 배포다.

`TLS termination(TLS 종료)`은 하나의 암호화 연결을 복호화하고 끝내는 지점을 뜻한다. 그 뒤에 새 TLS 또는 mTLS 연결을 다시 시작할 수 있으므로, TLS가 종료됐다고 해서 이후 모든 구간이 반드시 plaintext인 것은 아니다.

| 항목 | 기존 비교 기준선 | Istio 비교 환경 |
|---|---|---|
| 목적 | 기존 동작을 유지하는 대조군 | service mesh 동작을 검증하는 실험군 |
| public hostname | `task-api.lhrm-lab.com` | `task-api-istio.lhrm-lab.com` |
| 외부 진입점 | Traefik | Istio Gateway Envoy |
| namespace | `argo-task-api` | `argo-task-api-istio` |
| Task API Pod | 기존 별도 Pod, sidecar 없음 | 비교용 별도 Pod, Envoy sidecar 있음 |
| public TLS 종료 지점 | Traefik | Istio Gateway Envoy |
| proxy 이후 구간 | Traefik → Task API: HTTP | Gateway → Task API sidecar: 별도 mTLS |
| 완료 알림 | Task API가 Slack 직접 호출 | Task API → Notification Service → Slack |
| Kiali 가시성 | mesh 외부라 service 간 edge 없음 | Gateway 및 service 간 traffic 확인 가능 |

## 최종 구조와 TLS 종료 지점

기존 Traefik 경로에서는 공개 TLS가 Traefik에서 끝난다. Traefik이 요청을 복호화한 뒤 Task API에는 cluster 내부 HTTP로 전달한다. Task API가 Slack을 호출할 때에는 애플리케이션이 별도의 HTTPS 연결을 만들고, 이 연결은 Slack에서 끝난다.

```text
사용자
  == public HTTPS ==>
[Traefik: public TLS 종료]
  -- cluster HTTP -->
[argo-task-api Service]
  -- HTTP -->
[기존 Task API Pod: sidecar 없음]
  == 별도 HTTPS ==>
[Slack: HTTPS 종료]
```

Istio 경로에서는 공개 TLS가 Istio Gateway에서 끝난다. Gateway는 복호화한 요청을 그대로 plaintext로 보내지 않고, Task API sidecar와 **새로운 Istio mTLS 연결**을 만든다. sidecar가 mTLS를 끝내고 같은 Pod의 애플리케이션에는 local HTTP로 전달한다.

```text
사용자
  == public HTTPS ==>
[Istio Gateway Envoy: public TLS 종료]
  == 별도 Istio mTLS ==>
[Task API Envoy sidecar: mesh mTLS 종료]
  -- Pod 내부 local HTTP -->
[비교용 Task API]
  -- NotificationServiceClient의 HTTP 요청 -->
[Task API Envoy sidecar]
  == Istio mTLS ==>
[Notification Service Envoy sidecar: mesh mTLS 종료]
  -- Pod 내부 local HTTP -->
[Notification Service]
  == 애플리케이션이 만든 HTTPS, sidecar는 pass-through ==>
[Slack: HTTPS 종료]
```

즉, 기존 TLS termination을 Traefik에서 Istio Gateway로 **교체한 것이 아니다**. 기존 hostname과 Traefik 경로는 그대로 유지했고, Istio Gateway에서 TLS를 종료하는 새 hostname과 별도 배포 경로를 병렬로 추가했다. 또한 사용자부터 애플리케이션까지 하나의 TLS 연결이 이어지는 구조가 아니라, Gateway에서 public TLS가 끝나고 Gateway와 sidecar 사이에서 인증서 종류와 목적이 다른 mesh mTLS 연결이 새로 시작된다.

## 용어

### Workload

Kubernetes에서 실행되는 애플리케이션 단위다. 이 문서에서는 Task API와 Notification Service Deployment가 각각 별도 workload다.

### Sidecar

애플리케이션 container 옆에서 함께 실행되는 보조 proxy다. Istio의 Envoy sidecar는 들어오고 나가는 traffic을 대신 처리하고 metric을 기록한다.

### mTLS

Mutual TLS의 약자다. 통신 양쪽이 서로 인증서를 확인한 뒤 암호화해서 대화한다. 일반 TLS가 주로 서버만 증명한다면 mTLS는 client와 server가 모두 신원을 증명한다.

### PeerAuthentication

Istio workload가 plaintext를 받을 수 있는지, mTLS만 받을지 정하는 정책이다. `STRICT`는 mTLS가 아닌 연결을 거부한다.

### AuthorizationPolicy

mTLS로 확인한 workload 신원을 기준으로 어떤 요청을 허용할지 정하는 정책이다. 암호화되었다는 사실만으로 모든 요청을 허용하지 않는다.

### ServiceAccount와 principal

ServiceAccount는 Kubernetes workload의 신원이다. Istio는 이를 다음과 같은 principal 문자열로 표현한다.

```text
cluster.local/ns/<namespace>/sa/<service-account>
```

### ExternalSecret

AWS Secrets Manager 같은 외부 저장소의 값을 Kubernetes Secret으로 동기화하는 리소스다. Git에는 실제 webhook 값이 아니라 remote key와 property 이름만 남는다.

### ConfigMap checksum

ConfigMap 내용의 해시값을 Pod template annotation에 넣는 방식이다. 설정이 바뀌면 해시도 달라져 Kubernetes가 새 Pod를 자동으로 rollout한다.

## 핵심 설계 결정

### 기존 Traefik 환경을 유지한 이유

이번 작업의 목적은 기존 ingress를 Istio로 즉시 교체하는 것이 아니라, 기존 구조와 service mesh 구조를 같은 cluster에서 비교하는 것이다. 기존 경로까지 함께 변경하면 Gateway, sidecar, mTLS, Notification Service 중 무엇이 동작 차이를 만들었는지 구분하기 어려워진다. 따라서 Traefik 환경을 대조군으로 유지하고 Istio 환경을 별도 namespace와 hostname에 병렬 배포했다.

두 환경의 역할은 다음과 같이 분리했다.

- `argo-task-api`: `NOTIFIER=slack` 유지
- `argo-task-api-istio`: `NOTIFIER=service`로 변경
- 기존 public hostname과 Traefik 경로는 그대로 유지
- 인증 없는 `/tasks` public route는 추가하지 않음

이 구조를 선택한 이유는 다음과 같다.

- **비교 가능성**: sidecar가 없는 기존 흐름과 sidecar·mTLS가 있는 흐름의 latency, 오류율, traffic graph를 나란히 확인할 수 있다.
- **기존 경로 보호**: `task-api.lhrm-lab.com`의 DNS, 인증서, Traefik routing과 기존 호출 동작에 영향을 주지 않는다.
- **장애 범위 분리**: Istio 설정이나 Notification Service에 문제가 생겨도 기존 Task API의 직접 Slack 알림 경로는 계속 사용할 수 있다.
- **즉시 rollback**: 비교용 Istio Application만 중지하거나 제거하면 기존 경로로 돌아갈 수 있으며, ingress를 다시 Traefik으로 복구하는 작업이 필요 없다.
- **TLS 비교**: 기존 공개 TLS가 Traefik에서 종료되는 흐름과 공개 TLS가 Istio Gateway에서 종료된 뒤 mesh mTLS로 이어지는 흐름을 동시에 관찰할 수 있다.

Traefik을 반드시 계속 사용해야 하는 기술적 제약이 있는 것은 아니다. Istio 환경 검증 후 실제 진입 경로를 교체할지는 별도의 전환 작업으로 판단한다.

### 알림 실패가 task 완료를 되돌리지 않는다

Task 상태는 Notification Service를 호출하기 전에 DB에 commit된다. 내부 호출 또는 Slack 전송이 실패하면 로그와 metric에는 실패가 남지만 task 완료 상태는 유지된다.

## `task-api-platform` 파일별 변경

### `.env.example`

Task API가 Notification Service의 위치를 찾을 수 있도록 `NOTIFICATION_SERVICE_URL`을 Kubernetes service DNS 형식으로 문서화했다.

```text
http://notification-service.notification-service.svc.cluster.local:8000
```

### `app/services/notification_service.py`

알림을 Slack에 직접 보낼지 Notification Service에 전달할지 선택할 수 있도록 다음을 변경했다.

- `NotificationServiceClient`가 `POST /notifications/task-completed`를 호출한다.
- Python 표준 라이브러리 `urllib.request`를 재사용해 dependency를 추가하지 않았다.
- JSON body는 task의 `id`, `title`만 포함한다.
- timeout은 2초이며 자동 retry하지 않는다.
- `NOTIFIER=service`이면 내부 client를 선택한다.
- 내부 URL이 없으면 시작 자체를 깨지 않고 `NullNotifier`로 fallback하며 경고를 남긴다.
- 기존 `SlackNotifier`는 직접 호출 환경과 Notification Service 양쪽에서 재사용한다.
- Notification Service에서는 Slack 오류를 HTTP 5xx로 표현할 수 있도록 예외 전파 옵션을 사용한다.

### `app/notification_main.py`

Slack 전송을 담당하는 FastAPI 애플리케이션을 추가하고 다음 endpoint를 제공한다.

- `GET /healthz`: 프로세스 liveness 확인
- `GET /readyz`: Slack webhook 설정 존재 여부 확인
- `POST /notifications/task-completed`: 입력 검증 후 Slack 전송

요청의 `id`는 양수, `title`은 1~255자로 검증한다. Slack 성공은 `204`, webhook 미설정은 `503`, Slack 호출 실패는 `502`로 응답한다. DB와 Redis는 사용하지 않는다.

### `tests/test_notification.py`

다음 동작을 검증한다.

- 내부 client가 정확한 URL, JSON, timeout으로 호출하는지
- 내부 호출 오류가 caller에게 전달되는지
- `NOTIFIER=service` 선택과 URL 누락 fallback
- SlackNotifier의 기존 best-effort 동작과 Notification Service용 오류 전파 동작

### `tests/test_notification_api.py`

Notification Service의 health, readiness, 정상 `204`, 입력 오류 `422`, Slack 실패 `502`를 검증한다.

전체 애플리케이션 테스트 결과는 `43 passed`였다.

### `.github/workflows/ci-cd.yaml`

같은 image를 사용하는 세 workload가 서로 다른 버전을 실행하지 않도록 CI를 다음과 같이 변경했다.

- Python 3.12에서 `pytest -q`를 실행한다.
- 기존 Task API image tag를 갱신한다.
- 비교용 Istio Task API image tag도 함께 갱신한다.
- Notification Service Kustomize `newTag`가 존재하면 함께 갱신한다.
- 세 tag 변경을 한 GitOps commit으로 push한다.

최종 검증 image tag는 `8300b8e`다.

### `helm/task-api/templates/deployment.yaml`

ConfigMap checksum annotation을 Pod template에 추가했다.

```yaml
checksum/config: <rendered ConfigMap hash>
```

이 변경이 없으면 Argo CD가 `NOTIFIER=service`로 ConfigMap을 바꿔도 기존 Pod는 이전 환경변수를 계속 사용한다. checksum이 달라지면 Deployment가 자동 rollout되어 새 설정을 읽는다.

## `gitops-argocd` 파일별 변경

### `clusters/dev/applications/notification-service.yaml`

Notification Service 리소스를 관리하는 Argo CD child Application이다.

- source: `clusters/dev/notification-service`
- destination namespace: `notification-service`
- namespace에 `istio-injection=enabled` 적용
- auto sync, prune, self-heal 사용

### `clusters/dev/notification-service/kustomization.yaml`

Notification Service manifest를 묶고 image를 치환한다. CI가 `newTag`를 갱신하므로 Task API와 같은 image version을 사용한다.

### `clusters/dev/notification-service/deployment.yaml`

Notification Service Pod를 실행한다.

- replica 1개
- 기존 task-api image 재사용
- 실행 entrypoint를 `app.notification_main:app`으로 override
- Slack Secret을 `envFrom`으로 주입
- health와 readiness probe 사용
- 전용 ServiceAccount 사용
- capability 제거, privilege escalation 금지, seccomp 적용

초기에는 `runAsNonRoot: true`도 선언했지만 image의 `USER app`이 이름 기반이라 kubelet이 숫자 UID를 검증하지 못했다. image가 이미 non-root user를 사용하므로 이 중복 항목만 제거했고 나머지 보안 설정은 유지했다.

### `clusters/dev/notification-service/service.yaml`

cluster 내부에서만 접근할 수 있는 ClusterIP Service다. 8000번 포트 이름을 `http`로 지정해 Istio가 HTTP traffic으로 인식하게 한다.

### `clusters/dev/notification-service/serviceaccount.yaml`

Notification Service 전용 identity를 만든다.

```text
cluster.local/ns/notification-service/sa/notification-service
```

### `clusters/dev/secretops/externalsecret-notification-service.yaml`

기존 AWS Secrets Manager의 task completion Slack webhook을 Notification Service namespace로 동기화한다. DB와 Redis Secret은 포함하지 않는다.

### `clusters/dev/applications/argo-task-api-istio.yaml`

비교용 Task API 설정을 다음처럼 변경했다.

```text
NOTIFIER=service
NOTIFICATION_SERVICE_URL=http://notification-service.notification-service.svc.cluster.local:8000
```

기존 `argo-task-api` Application은 `NOTIFIER=slack`을 계속 사용한다.

### `clusters/dev/secretops/externalsecret-task-api-istio.yaml`

비교용 Task API Secret에서 `NOTIFY_WEBHOOK_URL` mapping을 제거했다. 최종 Secret에는 DB와 Redis password만 남는다. Slack credential은 Notification Service만 가진다.

### `clusters/dev/applications/istio-workload-policies.yaml`

두 namespace의 Istio 보안 정책을 함께 관리하는 Argo CD child Application이다. 정책 manifest가 여러 namespace에 있으므로 destination namespace를 고정하지 않는다.

### `clusters/dev/istio/policies/*-peerauthentication.yaml`

`argo-task-api-istio`와 `notification-service` namespace에 각각 `STRICT` mTLS를 적용한다.

```yaml
spec:
  mtls:
    mode: STRICT
```

sidecar 없는 workload의 plaintext 연결은 TLS handshake 전에 reset된다.

### `clusters/dev/istio/policies/argo-task-api-istio-authorizationpolicy.yaml`

실제 Gateway Pod의 ServiceAccount를 조회한 뒤 principal을 고정했다.

```text
cluster.local/ns/istio-ingress/sa/task-api-istio-gateway-istio
```

이 principal이 Task API 8000번 포트의 다음 GET 경로를 호출하는 것만 허용한다.

- `/version`
- `/healthz`
- `/readyz`

### `clusters/dev/istio/policies/notification-service-authorizationpolicy.yaml`

비교용 Task API의 실제 identity만 알림 endpoint를 호출할 수 있다.

```text
cluster.local/ns/argo-task-api-istio/sa/default
```

Task API Helm chart에 ServiceAccount 연결 template가 없으므로 비교 namespace의 `default` ServiceAccount를 사용한다. 허용 범위는 8000번 포트의 `POST /notifications/task-completed`다.

## 실제 검증 결과

| 검증 | 결과 |
|---|---|
| Argo CD Applications | `Synced / Healthy` |
| Task API Pod | 앱 + Envoy `2/2 Running` |
| Notification Service Pod | 앱 + Envoy `2/2 Running` |
| Gateway → Task API | HTTPS `/version` 성공 |
| Task 완료 → 내부 서비스 → Slack | 성공, Notification Service `204` |
| Istio telemetry | `argo-task-api-istio -> notification-service`, destination reporter `mutual_tls` |
| STRICT plaintext 차단 | sidecar 없는 기존 Task API 요청이 connection reset |
| 미인가 mTLS 차단 | Notification Service identity의 두 대상 요청이 `403 Forbidden` |
| 허용된 identity | Task API principal의 알림 POST 성공 |
| 테스트 데이터 | 검증 task 삭제 완료 |

source reporter의 `connection_security_policy`는 `unknown`일 수 있다. 실제 수신 연결을 판정하는 destination reporter에서 `mutual_tls`을 확인했다.

## Kiali에서 확인하는 방법

1. Kiali Traffic Graph를 연다.
2. namespace에서 `argo-task-api-istio`, `notification-service`를 함께 선택한다.
3. 시간 범위를 최근 5~10분으로 둔다.
4. 아래 task 완료 요청을 실행한다.
5. `argo-task-api-istio -> notification-service` edge와 lock/mTLS 표시, 요청량, latency, 오류율을 확인한다.

```bash
kubectl -n argo-task-api-istio port-forward svc/argo-task-api-istio 18000:8000
```

다른 terminal에서 실행한다.

```bash
TASK_ID=$(curl -sS -X POST http://127.0.0.1:18000/tasks \
  -H 'Content-Type: application/json' \
  -d '{"title":"kiali internal traffic test"}' | jq -r '.id')

curl -sS -X PATCH "http://127.0.0.1:18000/tasks/${TASK_ID}" \
  -H 'Content-Type: application/json' \
  -d '{"done":true}'

curl -i -X DELETE "http://127.0.0.1:18000/tasks/${TASK_ID}"
```

## 정책 확인 명령

```bash
kubectl get peerauthentication -A
kubectl get authorizationpolicy -A
kubectl get pods -n argo-task-api-istio
kubectl get pods -n notification-service
```

plaintext 차단은 sidecar 없는 기존 Task API에서 확인할 수 있다.

```bash
kubectl exec -n argo-task-api deployment/argo-task-api -c api -- \
  python -c 'import urllib.request; urllib.request.urlopen(
    "http://notification-service.notification-service.svc.cluster.local:8000/healthz",
    timeout=3,
  )'
```

예상 결과는 connection reset과 non-zero exit code다.

## Rollback 순서

1. 두 AuthorizationPolicy를 제거한다.
2. 두 PeerAuthentication `STRICT`를 제거한다.
3. 비교용 Task API ExternalSecret에 Slack webhook mapping을 복구한다.
4. 비교용 Task API를 `NOTIFIER=slack`으로 되돌린다.
5. rollout과 Slack 직접 호출을 확인한다.
6. Notification Service Application과 ExternalSecret을 제거한다.

기존 Traefik Task API는 rollback 대상이 아니다.

## 의도적으로 제외한 범위

- queue, Kafka/RabbitMQ
- 자동 retry
- transactional outbox
- 별도 Notification repository와 ECR
- public `/tasks` route
- mesh 전체에 적용하는 global STRICT
- RDS, Redis, Slack egress의 별도 Istio 정책
- Python HTTP client의 trace context 전파

알림의 보장 전달이나 서비스별 독립 배포가 실제 요구사항이 될 때 queue/outbox 또는 별도 artifact를 검토한다.

## 보안 후속 조치

검증 과정에서 기존 Slack webhook 값이 작업 transcript에 base64 형태로 노출되었다. Slack webhook을 재발급하고 AWS Secrets Manager의 `TASK_COMPLETION_WEBHOOK_URL`을 교체해야 한다. Git 저장소에는 webhook 값이 들어 있지 않다.

## 주요 작업 commit

### `task-api-platform`

- `e458dd4` — Notification Service endpoint와 client
- `4970877` — 테스트 및 세 workload image tag 동기화 CI
- `e13f102` — ConfigMap 변경 시 자동 rollout

### `gitops-argocd`

- `4860c3e` — Notification Service 배포 리소스
- `f1715cb` — 이름 기반 non-root image 실행 수정
- `eba7de5` — Istio Task API 내부 알림 라우팅
- `e85c3f7` — 두 namespace STRICT mTLS
- `ea29306` — workload identity AuthorizationPolicy
