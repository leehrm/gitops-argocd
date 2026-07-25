# OpenTelemetry tracing

## 1. 목적과 범위

Task API의 HTTP 요청을 trace로 수집하여 FastAPI, Redis, PostgreSQL 구간의 처리 순서, 지연, 오류를 Grafana에서 분석한다.

이 변경의 범위는 다음 tracing 인프라이다.

- OpenTelemetry Collector: OTLP trace 수신, processor 처리, Tempo로 export
- Tempo: trace 저장 및 TraceQL 조회 API 제공
- Grafana: Tempo datasource provisioning 및 Loki trace-to-logs 연결 준비

Task API의 OpenTelemetry SDK, instrumentation, Helm 환경변수는 `task-api-platform` 저장소의 후속 작업에서 구성한다. 따라서 현재 단계에서 Tempo에 Task API trace가 없어도 정상이다.

## 2. 아키텍처

```text
Task API OpenTelemetry SDK
  -- OTLP/gRPC 4317 or OTLP/HTTP 4318 -->
OpenTelemetry Collector
  -- OTLP/gRPC 4317 -->
Tempo
  <-- HTTP 3200 --
Grafana
```

Kubernetes Service endpoint는 다음과 같다.

| 구간 | Endpoint |
|---|---|
| Task API → Collector OTLP/gRPC | `opentelemetry-collector.tracing.svc.cluster.local:4317` |
| Task API → Collector OTLP/HTTP | `http://opentelemetry-collector.tracing.svc.cluster.local:4318` |
| Collector → Tempo OTLP/gRPC | `tempo.tracing.svc.cluster.local:4317` |
| Grafana → Tempo query API | `http://tempo.tracing.svc.cluster.local:3200` |

모든 Service는 `ClusterIP`이며 Ingress는 생성하지 않는다.

## 3. Trace 정책

### 3.1 `/tasks` span 설계

| API | Repository 처리 | Cache 처리 | Custom attributes |
|---|---|---|---|
| `POST /tasks` | Primary `INSERT ... RETURNING` | `tasks:list` invalidate | `task.operation=create`, `db.role=write`, `db.target=primary` |
| `GET /tasks` | cache miss 시 Replica `SELECT` | `tasks:list` get/set | `task.operation=list`, `db.role=read`, `db.target=replica` |
| `GET /tasks/{task_id}` | cache miss 시 Replica `SELECT` | `tasks:item:{task_id}` get/set | `task.operation=get`, `db.role=read`, `db.target=replica` |
| `PATCH /tasks/{task_id}` | Primary `UPDATE ... RETURNING` | list/item invalidate | `task.operation=update`, `db.role=write`, `db.target=primary` |
| `DELETE /tasks/{task_id}` | Primary `DELETE ... RETURNING` | list/item invalidate | `task.operation=delete`, `db.role=write`, `db.target=primary` |

`deploy/dev`의 `DB_PRIMARY_HOST`와 `DB_READ_HOST`는 서로 다른 RDS endpoint이다. 따라서 dev 환경에서 `db.target=primary|replica`를 구분할 수 있다.

쓰기 API는 Primary의 `RETURNING` 결과를 응답하며 추가 read query를 실행하지 않는다. 쓰기 직후 별도의 GET 요청은 Replica를 사용한다. Replica 연결 또는 query 실패 시 Primary로 runtime fallback하지 않는다.

### 3.2 Redis span 설계

캐시는 cache-aside 패턴을 사용하며 기본 TTL은 60초이다. Redis instrumentation이 생성하는 client span에 다음 custom attributes를 추가한다.

```text
cache.operation = get | set | delete
cache.hit = true | false
cache.key_pattern = tasks:list | tasks:item:{task_id}
```

`cache.key_pattern`은 실제 `task_id`를 제거한 low-cardinality 값을 사용한다. `cache.hit`는 GET span에만 추가한다.

Redis GET, SETEX, DEL, JSON decode 실패는 현재 HTTP 요청을 실패시키지 않는다. Redis 오류 후 DB fallback이 성공하면 span status는 다음과 같아야 한다.

```text
HTTP server span: UNSET
Redis client span: ERROR
DB client span: UNSET
```

OpenTelemetry에서 성공 span의 status는 일반적으로 `OK`를 명시하기보다 `UNSET`을 유지한다. Redis cache invalidation 실패로 stale cache가 TTL까지 남는 현재 동작은 변경하지 않고 Redis span error로 관측한다.

### 3.3 HTTP 상태와 span status

| 상황 | HTTP 상태 | Server span status |
|---|---:|---|
| Task가 없음 | 404 | `UNSET` |
| FastAPI request validation 실패 | 422 | `UNSET` |
| `/internal/drain` 접근 거부 | 403 | `UNSET` |
| PostgreSQL 연결/query 실패 | 500 | `ERROR` |
| Redis 오류 후 DB fallback 성공 | 200 | `UNSET` |
| Unhandled application exception | 500 | `ERROR` |
| `/debug/error` | 500 | `ERROR` |

HTTP server span은 5xx를 `ERROR`로 판정한다. 4xx는 server 관점에서 처리된 client request이므로 `UNSET`을 유지한다.

### 3.4 지연 기준

| Span | 초기 기준 |
|---|---:|
| HTTP server | 500ms 이상 |
| PostgreSQL client | 200ms 이상 |
| Redis client | 100ms 이상 |
| `GET /debug/slow` | `DEBUG_SLOW_SECONDS` |

이 기준은 span status를 `ERROR`로 변경하는 조건이 아니라 지연 trace 검색·분석 기준이다. 현재 dev의 `DEBUG_SLOW_SECONDS=0.3`은 HTTP 500ms 기준보다 짧으므로 지연 검증 시 0.6초 이상으로 임시 조정한다.

### 3.5 민감정보와 cardinality

다음 값은 span attributes, events, baggage에 기록하지 않는다.

- DB·Redis credential
- `Authorization`, `Cookie` 헤더
- Task title 및 request/response body
- SQL bind parameter
- `task.id`
- `task_id`가 포함된 Redis key 원문

### 3.6 Trace 제외 URL

반복 호출되는 health check, metrics, API schema, lifecycle URL은 FastAPI instrumentation의 excluded URLs로 설정한다.

```text
/metrics
/healthz
/readyz
/docs
/redoc
/openapi.json
/static/*
/internal/drain
```

`/version`은 배포 버전 확인 trace로 활용할 수 있으므로 제외하지 않는다.

### 3.7 Sampling

dev는 instrumentation 검증을 위해 `parentbased_always_on`, 즉 100% head sampling을 사용한다. 운영 sampling 비율과 Collector tail sampling은 이번 범위에서 제외한다.

## 4. GitOps 구성

### 4.1 Tempo

| 항목 | 설정 |
|---|---|
| Argo CD Application | `tempo` |
| Namespace | `tracing` |
| Helm Chart | `grafana/tempo` `1.24.4` |
| Image | `grafana/tempo:2.9.0` |
| Workload | StatefulSet, replica 1 |
| Storage backend | local filesystem |
| Persistence | RWO PVC 5Gi, default StorageClass |
| Retention | 24h |
| Resource requests | CPU 250m, memory 512Mi |
| Resource limits | CPU 1, memory 1Gi |
| Service | ClusterIP |
| Argo CD sync wave | 1 |

Tempo Chart는 OTLP 외에 Jaeger receiver와 호환용 Service port를 렌더링한다. 외부 Ingress와 LoadBalancer를 생성하지 않으므로 cluster 외부에서는 접근할 수 없다. 이 구성에서 사용하는 ingestion protocol은 OTLP이다.

### 4.2 OpenTelemetry Collector

| 항목 | 설정 |
|---|---|
| Argo CD Application | `opentelemetry-collector` |
| Namespace | `tracing` |
| Helm Chart | `open-telemetry/opentelemetry-collector` `0.165.0` |
| Image | `opentelemetry-collector-k8s:0.156.0` |
| Workload | Deployment, replica 1 |
| Service | ClusterIP, OTLP 4317/4318 |
| Resource requests | CPU 100m, memory 128Mi |
| Resource limits | CPU 500m, memory 512Mi |
| Argo CD sync wave | 2 |

Collector는 `alternateConfig`를 사용해 Chart의 기본 logs, metrics, Jaeger, Zipkin pipeline을 제거하고 traces pipeline만 구성한다.

```text
otlp receiver
  -> memory_limiter processor
  -> batch processor
  -> otlp_grpc/tempo exporter
```

`health_check` extension은 Kubernetes liveness/readiness probe를 위해 유지한다. `memory_limiter`는 container memory limit의 80%를 soft limit, 25%를 spike limit으로 사용한다.

### 4.3 Grafana datasource

Grafana sidecar가 `grafana_datasource: "1"` label이 있는 ConfigMap을 provisioning한다.

- Tempo datasource UID: `tempo`
- Loki datasource UID: `loki`
- Tempo URL: `http://tempo.tracing.svc.cluster.local:3200`
- Trace-to-logs datasource: `loki`
- Trace-to-logs time range: span start -1m, span end +1m
- Resource attribute → Loki label mapping: `service.name → app`, `k8s.namespace.name → namespace`

Trace-to-logs는 Task API log record에 `trace_id`가 포함되고, Tempo resource attributes와 Loki labels가 일치한 뒤 완전하게 동작한다.

## 5. 배포 전 검증 결과

로컬에서 고정한 Chart 버전으로 다음을 검증했다.

- Tempo, Collector `helm lint` 성공
- Tempo, Collector `helm template` 성공
- Argo CD Application과 Grafana ConfigMap YAML parsing 성공
- Tempo Service에 3200, 4317, 4318 port 존재
- Collector Service에 4317, 4318 port만 존재
- Collector exporter endpoint와 Tempo Service DNS 일치
- Tempo StatefulSet에 5Gi volumeClaimTemplate 존재
- PVC의 `storageClassName` 미지정
- Ingress 및 LoadBalancer Service 미생성

Collector Chart `0.165.0`은 공식 repository의 최신 버전이지만 Helm lint에서 deprecation warning이 발생한다. lint와 rendering은 성공했으며, 후속 업그레이드 시 공식 유지보수 정책과 대체 배포 방식을 재확인한다.

## 6. EKS 배포 후 검증

### 6.1 Argo CD 상태

```bash
kubectl get applications -n argocd
```

`tempo`, `opentelemetry-collector`, `task-api-monitoring`이 `Synced/Healthy`인지 확인한다.

### 6.2 Workload, Service, PVC

```bash
kubectl get pods,deploy,statefulset,svc,pvc -n tracing
kubectl get storageclass
kubectl describe pvc -n tracing
```

예상 결과:

- Tempo StatefulSet/Pod: 1 replica, `Running`
- Collector Deployment/Pod: 1 replica, `Running`
- Tempo PVC: `Bound`, 5Gi
- Tempo/Collector Service: `ClusterIP`
- 기본 StorageClass provisioner: `ebs.csi.aws.com`

### 6.3 Application log

```bash
kubectl logs -n tracing deploy/opentelemetry-collector
kubectl logs -n tracing -l app.kubernetes.io/name=tempo
```

Collector startup config error, Tempo DNS resolution error, OTLP exporter retry가 반복되지 않는지 확인한다.

### 6.4 Grafana

Grafana의 **Connections > Data sources**에서 `Tempo`를 확인하고 datasource health check를 실행한다. Task API instrumentation 전에는 Explore 결과가 비어 있을 수 있다.

### 6.5 OTLP smoke test

Task API instrumentation 전에는 임시 OTLP trace generator로 다음 경로를 검증한다.

```text
trace generator -> Collector -> Tempo -> Grafana Explore
```

검증 후 임시 Pod을 삭제한다. 테스트 Pod은 GitOps manifest에 포함하지 않는다.

## 7. Task API instrumentation 후 검증 시나리오

1. `GET /tasks` 첫 호출: cache miss, Redis GET, Replica SELECT, Redis SETEX span 확인
2. `GET /tasks` 재호출: cache hit, DB span이 생성되지 않음을 확인
3. `POST /tasks`: Primary INSERT, `tasks:list` DEL span 확인
4. `PATCH /tasks/{task_id}`: Primary UPDATE, list/item DEL span 확인
5. `DELETE /tasks/{task_id}`: Primary DELETE, list/item DEL span 확인
6. 없는 Task 조회: HTTP 404, server span status `UNSET` 확인
7. `/debug/error`: HTTP 500, server span status `ERROR` 확인
8. `/debug/slow`: 500ms 이상 trace 검색 확인
9. Redis 장애: Redis span `ERROR`, DB fallback 성공, server span `UNSET` 확인
10. `/metrics`, `/healthz`, `/readyz` 등 excluded URL에서 span이 생성되지 않음을 확인
11. Grafana의 trace-to-logs link가 같은 `trace_id`의 Loki log를 조회하는지 확인

## 8. 후속 작업

`task-api-platform`에서 다음 순서로 진행한다.

1. OpenTelemetry Python SDK, OTLP exporter, FastAPI·HTTPX·Redis·PostgreSQL instrumentation 패키지 추가
2. `TracerProvider`, resource attributes, `BatchSpanProcessor`, OTLP exporter 구성
3. FastAPI excluded URLs 설정
4. cache/DB custom attributes 추가
5. application log에 `trace_id`, `span_id` correlation 추가
6. Helm Chart에 `OTEL_SERVICE_NAME`, `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_TRACES_SAMPLER` 등 환경변수 추가
7. local integration test 및 Helm rendering 검증
8. EKS 배포 후 7장의 시나리오 실행

Istio, Kiali, Tempo distributed mode, S3/IRSA, metrics-generator, Service Graph, production sampling은 이번 범위에서 제외한다.
