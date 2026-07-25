# SecretOps 인벤토리 및 전환 기록

`deploy/dev` 환경의 SecretOps 도입 전 구조와 전환 완료 상태를 함께 기록한 문서.

## 현재 운영 상태

현재 credential 공급 경로는 아래 방식으로 통일 완료.

```text
AWS Secrets Manager
→ External Secrets Operator
→ Kubernetes Secret
→ workload
```

| 항목 | AWS Secrets Manager key / property | ExternalSecret | Kubernetes Secret | 소비 workload | 상태 |
|---|---|---|---|---|---|
| task-api DB credential | `/aws-eks-terraform-lab/dev/task-api/database` / `DB_PASSWORD` | `task-api-runtime-secrets` | `argo-task-api/task-api-runtime-secrets` / `DB_PASSWORD` | `argo-task-api` Deployment | 전환 완료. |
| task-api Redis credential | `/aws-eks-terraform-lab/dev/redis/auth` / `REDIS_PASSWORD` | `task-api-runtime-secrets` | `argo-task-api/task-api-runtime-secrets` / `REDIS_PASSWORD` | `argo-task-api` Deployment | 전환 완료. |
| Redis 서버 인증 | `/aws-eks-terraform-lab/dev/redis/auth` / `REDIS_PASSWORD` | `redis-auth` | `redis/redis-auth` / `redis-password` | Bitnami Redis workload | 전환 완료. |
| Grafana 관리자 credential | `/aws-eks-terraform-lab/dev/observability/grafana` / `ADMIN_USER`, `ADMIN_PASSWORD` | `grafana-admin-credentials` | `monitoring/grafana-admin-credentials` / `admin-user`, `admin-password` | Grafana workload | 전환 완료. |
| Alertmanager Slack webhook | `/aws-eks-terraform-lab/dev/observability/slack` / `WEBHOOK_URL` | `alertmanager-slack-webhook` | `monitoring/alertmanager-slack-webhook` / `slack-webhook-url` | AlertmanagerConfig `task-api-slack` | 전환 완료. |

## 전환 완료 검증

- AWS Secrets Manager에서 ESO를 통한 Kubernetes Secret 동기화 완료.
- task-api `/healthz` 응답 200 확인 완료.
- task-api `/readyz` 응답 200 및 DB connected 확인 완료.
- Redis `redis-cli ping` 결과 PONG 확인 완료.
- Grafana 관리자 로그인 확인 완료.
- Slack 테스트 Alert의 `#task-api-alerts` 수신 확인 완료.
- 관련 Argo CD Application의 Synced / Healthy 확인 완료.

## SecretOps 도입 전 인벤토리

아래 표는 도입 전 plaintext 위치와 기존 전달 구조를 남긴 과거 기록. `도입 전 전달 방식`은 인벤토리 조사 당시 상태.

| 용도 | 정의 파일 / YAML key 경로 | Namespace | 소비자 | 도입 전 전달 방식 | 분류 | 전환 target | 영향 저장소 | 당시 계획 / Rollback |
|---|---|---|---|---|---|---|---|---|
| task-api DB 비밀번호 | `clusters/dev/applications/argo-task-api.yaml`의 `spec.source.helm.values.secretEnv.DB_PASSWORD` | `argo-task-api` | `argo-task-api` Deployment, `task-api-platform` DB 설정 | Argo CD inline Helm values → Chart가 `argo-task-api-secret` 생성 → Deployment `envFrom.secretRef` | 민감정보 | `task-api-runtime-secrets` / `DB_PASSWORD` | `gitops-argocd`, `task-api-platform`, `aws-eks-terraform-lab` | ExternalSecret 준비 → Chart existing Secret 지원 → Deployment 전환 → inline 값 제거 작업 |
| task-api DB 사용자명 | 같은 파일의 `spec.source.helm.values.env.DB_USER` | `argo-task-api` | `argo-task-api` Deployment | ConfigMap을 통한 일반 환경변수 | 일반 설정·credential 식별자 | ConfigMap 유지 | `gitops-argocd`, `task-api-platform` | 비밀번호와 분리하여 일반 설정으로 유지 |
| task-api Redis 비밀번호 | `clusters/dev/applications/argo-task-api.yaml`의 `spec.source.helm.values.secretEnv.REDIS_PASSWORD` | `argo-task-api` | `argo-task-api` Deployment, cache 설정 | inline Helm values → `argo-task-api-secret` → Deployment `envFrom.secretRef` | 민감정보 | `task-api-runtime-secrets` / `REDIS_PASSWORD` | 세 저장소 모두 | Redis 서버 Secret 준비 후 task-api 소비자 전환 작업 |
| Redis 서버 인증 | `clusters/dev/applications/redis.yaml`의 `spec.source.helm.values.auth.password` | `redis` | Bitnami Redis Chart, Redis Pod 및 task-api 클라이언트 | Argo CD inline values → Bitnami Chart 생성 Secret → Redis Pod | 민감정보 | `redis-auth` / `redis-password` | `gitops-argocd`, `task-api-platform`, `aws-eks-terraform-lab` | `auth.existingSecret`, `auth.existingSecretPasswordKey` 방식으로 전환 작업 |
| Grafana 관리자 계정 | `clusters/dev/applications/kube-prometheus-stack.yaml`의 `spec.source.helm.values.grafana.adminUser`, `grafana.adminPassword` | `monitoring` | kube-prometheus-stack의 Grafana subchart/Grafana Pod | inline Helm values → Grafana subchart가 관리자 Secret 생성 | 사용자명은 일반 설정, 비밀번호는 민감정보 | `grafana-admin-credentials` / `admin-user`, `admin-password` | `gitops-argocd`, `aws-eks-terraform-lab` | `grafana.admin.existingSecret` 방식으로 전환 작업 |
| Alertmanager Slack webhook | `clusters/dev/monitoring/alertmanager/alertmanager-slack-secret.example.yaml`의 `stringData.slack-webhook-url` | `monitoring` | `clusters/dev/monitoring/task-api/alertmanagerconfig-slack.yaml`의 `spec.receivers[].slackConfigs[].apiURL.name/key` | 예시 Secret 파일만 존재했으며 Child Application source에 포함되지 않았음 | 민감정보 | `alertmanager-slack-webhook` / `slack-webhook-url` | `gitops-argocd`, `aws-eks-terraform-lab` | 동일 이름과 key를 가진 ExternalSecret 전환 작업 |
| Let’s Encrypt staging 계정 private key | `clusters/dev/cert-manager/clusterissuers.yaml`의 staging `spec.acme.privateKeySecretRef.name` | cert-manager cluster resource namespace | cert-manager ClusterIssuer | cert-manager가 지정된 Secret 생성·관리 | 민감한 private key, Git plaintext 아님 | cert-manager 관리 유지 | `gitops-argocd` | SecretOps 이전 대상에서 제외 |
| Let’s Encrypt production 계정 private key | 같은 파일의 production `spec.acme.privateKeySecretRef.name` | 동일 | production ClusterIssuer | cert-manager 자동 생성·관리 | 민감한 private key, Git plaintext 아님 | cert-manager 관리 유지 | 동일 | SecretOps 이전 대상에서 제외 |
| task-api TLS private key·인증서 | `clusters/dev/edge/task-api/ingress.yaml`의 `spec.tls[].secretName` | `argo-task-api` | HTTPS Ingress, Traefik | cert-manager가 Ingress annotation 기준으로 TLS Secret 생성 | private key는 민감정보, 인증서는 공개정보 | cert-manager 관리 유지 | `gitops-argocd` | SecretOps 이전 대상에서 제외 |

## 도입 전 추가 확인 결과

- `clusters/dev`에서 별도 AWS access key, API token, bearer token plaintext 정의는 발견하지 못했음.
- task-api Redis credential과 Redis Chart auth가 서로 다른 Application values에 중복 정의되어 있었음.
- `task-api-platform`의 로컬 `.env`와 Helm values에도 관련 credential key가 존재했음.
- task-api Chart 초기 구조는 `secretEnv`를 이용해 Chart가 Secret을 직접 생성하는 방식이었음.
- task-api Chart `0.1.1`에 existing Secret 지원 추가 후 `task-api-runtime-secrets`로 workload cutover 완료.

## 전환 작업 기록

1. AWS Secrets Manager 경로와 property 정의 완료.
2. ESO 및 IRSA 구성 완료.
3. ClusterSecretStore와 ExternalSecret 배포 완료.
4. Alertmanager Slack webhook 전환 완료.
5. Grafana 관리자 credential 전환 완료.
6. Redis 서버 auth existing Secret 전환 완료.
7. task-api Chart existing Secret 지원 및 DB/Redis credential 전환 완료.
8. GitOps Application의 active plaintext credential 제거 완료.
9. cert-manager ACME/TLS key는 기존 cert-manager 관리 방식 유지.

## 후속 작업

### Credential rotation

- Git history에 노출된 기존 credential rotation 필요.
- AWS Secrets Manager의 새 버전 등록 후 ESO 동기화와 workload 재검증 작업.
- Redis 서버와 task-api Redis credential은 동일 rotation 단계로 처리하는 작업.

### gitleaks

- 현재 작업 트리와 Git history 대상 gitleaks 점검 필요.
- pull request 단계에서 plaintext credential 재유입을 차단하는 CI 검사 추가 작업.
- 탐지 결과의 허용 목록은 실제 비밀값이 아닌 명확한 테스트 fixture만 대상으로 관리하는 작업.

## Rollback 기준

- plaintext inline values 복원은 사용하지 않음.
- AWS Secrets Manager의 직전 정상 버전 복구 후 ESO 재동기화 방식 사용.
- target Kubernetes Secret과 workload 참조 이름은 유지하는 방식으로 rollback 수행.
