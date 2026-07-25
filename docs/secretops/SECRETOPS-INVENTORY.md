# SecretOps 도입 전 인벤토리

인벤토리 조사 기준 브랜치는 `deploy/dev`이며 조사 당시 작업 트리는 clean 상태였다. 파일 변경, commit, push, Argo CD sync, kubectl 변경은 수행하지 않았으며 Secret 값은 출력·마스킹·비교 표시하지 않았다.

## SecretOps 인벤토리

| 용도 | 정의 파일 / YAML key 경로 | Namespace | 소비자 | 현재 전달 방식 | 분류 | AWS Secrets Manager 이전 시 Kubernetes Secret | 영향 저장소 | 전환 순서 / Rollback |
|---|---|---|---|---|---|---|---|---|
| task-api DB 비밀번호 | `clusters/dev/applications/argo-task-api.yaml`의 `spec.source.helm.values.secretEnv.DB_PASSWORD` | `argo-task-api` | `argo-task-api` Deployment, `task-api-platform` DB 설정 | Argo CD inline Helm values → Chart가 `argo-task-api-secret` 생성 → Deployment `envFrom.secretRef` | 민감정보 | 권장: `task-api-runtime-secrets` / `DB_PASSWORD`; 기존 이름 유지 시 `argo-task-api-secret` / `DB_PASSWORD` | `gitops-argocd`, `task-api-platform`, `aws-eks-terraform-lab`(ASM·IAM) | ExternalSecret 준비 → Chart에 existing Secret 지원 → Deployment 전환 → inline 값 제거. Rollback: 기존 Chart 생성 Secret과 inline key 복원 |
| task-api DB 사용자명 | 같은 파일의 `spec.source.helm.values.env.DB_USER` | `argo-task-api` | `argo-task-api` Deployment | ConfigMap을 통한 일반 환경변수 | 일반 설정·credential 식별자 | 선택적으로 `task-api-runtime-secrets` / `DB_USER`; 최소 이전에서는 ConfigMap 유지 가능 | `gitops-argocd`, `task-api-platform` | 비밀번호와 동시에 옮길 경우 애플리케이션 환경변수 이름 유지. Rollback: ConfigMap key 복원 |
| task-api Redis 비밀번호 | `clusters/dev/applications/argo-task-api.yaml`의 `spec.source.helm.values.secretEnv.REDIS_PASSWORD` | `argo-task-api` | `argo-task-api` Deployment, cache 설정 | inline Helm values → `argo-task-api-secret` → Deployment `envFrom.secretRef` | 민감정보 | `task-api-runtime-secrets` / `REDIS_PASSWORD` | 세 저장소 모두 | Redis 서버 쪽 Secret을 먼저 준비한 뒤 task-api 소비자를 전환. Rollback: 이전 Redis credential이 유효한 동안 기존 Secret 참조 복원 |
| Redis 서버 인증 | `clusters/dev/applications/redis.yaml`의 `spec.source.helm.values.auth.password` | `redis` | Bitnami Redis Chart, Redis Pod 및 task-api 클라이언트 | Argo CD inline values → Bitnami Chart 생성 Secret → Redis Pod | 민감정보 | 권장: `redis-auth` / `redis-password`; values는 `auth.existingSecret`, `auth.existingSecretPasswordKey` 사용 | `gitops-argocd`, `task-api-platform`, `aws-eks-terraform-lab` | `redis-auth` 생성 → Chart를 existing Secret으로 전환 → 연결 확인 → task-api 전환 → 마지막에 회전. Rollback: 이전 password와 Chart 설정 복원. Bitnami Chart가 existing Secret을 지원함은 [공식 values](https://github.com/bitnami/charts/blob/main/bitnami/redis/values.yaml)에서 확인됨 |
| Grafana 관리자 계정 | `clusters/dev/applications/kube-prometheus-stack.yaml`의 `spec.source.helm.values.grafana.adminUser`, `grafana.adminPassword` | `monitoring` | kube-prometheus-stack의 Grafana subchart/Grafana Pod | inline Helm values → Grafana subchart가 관리자 Secret 생성 | 사용자명은 일반 설정, 비밀번호는 민감정보 | `grafana-admin-credentials` / `admin-user`, `admin-password` | `gitops-argocd`, `aws-eks-terraform-lab` | ExternalSecret 생성 → `grafana.admin.existingSecret` 및 user/password key 지정 → 로그인 확인 → inline 값 제거. Rollback: 기존 `adminUser`/`adminPassword` 방식 복원. 현재 stack `86.1.1`은 Grafana chart `12.4.2`를 사용함([Chart.yaml](https://raw.githubusercontent.com/prometheus-community/helm-charts/kube-prometheus-stack-86.1.1/charts/kube-prometheus-stack/Chart.yaml)) |
| Alertmanager Slack webhook | `clusters/dev/monitoring/alertmanager/alertmanager-slack-secret.example.yaml`의 `stringData.slack-webhook-url` | `monitoring` | `clusters/dev/monitoring/task-api/alertmanagerconfig-slack.yaml`의 `spec.receivers[].slackConfigs[].apiURL.name/key` | 예시 Secret 파일만 존재. 해당 경로는 Child Application source에 포함되지 않음 | 민감정보 | 기존 참조 그대로 `alertmanager-slack-webhook` / `slack-webhook-url` | `gitops-argocd`, `aws-eks-terraform-lab` | ExternalSecret으로 동일 이름/key 생성 → AlertmanagerConfig 상태 확인. Rollback: 이전 Secret을 안전한 방식으로 재생성. 현재는 desired state만으로 참조가 충족되지 않음 |
| Prometheus·Alertmanager BasicAuth | `clusters/dev/secretops/externalsecret-observability-basic-auth.yaml`의 `USERS` → `users` | `monitoring` | Traefik BasicAuth Middleware | AWS Secrets Manager → ExternalSecret | 민감정보 | `observability-basic-auth` / `users` | `gitops-argocd`, `aws-eks-terraform-lab` (`/aws-eks-terraform-lab/dev/observability/basic-auth`) | ASM 컨테이너와 값을 먼저 준비한 뒤 ExternalSecret, Middleware, Ingress 순으로 적용. Rollback: platform-edge 제거 후 남은 Secret 수동 삭제 |
| Let’s Encrypt staging 계정 private key | `clusters/dev/cert-manager/clusterissuers.yaml`의 staging `spec.acme.privateKeySecretRef.name` | 일반적으로 cert-manager의 cluster resource namespace | cert-manager ClusterIssuer | cert-manager가 지정된 Secret을 생성·관리 | 민감한 private key, Git 평문 아님 | ASM 이전 비권장. Secret 이름은 manifest에 지정되어 있으나 key는 저장소에 명시되지 않음 | `gitops-argocd`; cert-manager 설치 책임 저장소 | cert-manager 소유권 유지 권장. 이전 시 계정 등록·갱신 영향을 별도 검증. Rollback은 기존 cert-manager 관리 Secret 복원 |
| Let’s Encrypt production 계정 private key | 같은 파일의 production `spec.acme.privateKeySecretRef.name` | 동일 | production ClusterIssuer | cert-manager 자동 생성·관리 | 민감한 private key, Git 평문 아님 | ASM 이전 비권장; 이름만 manifest에 지정되고 key는 명시되지 않음 | 동일 | staging과 분리하여 처리. 잘못 교체하면 인증서 발급 계정 연속성이 손실될 수 있음 |
| task-api TLS private key·인증서 | `clusters/dev/edge/task-api/ingress.yaml`의 `spec.tls[].secretName` | `argo-task-api` | HTTPS Ingress, Traefik | cert-manager가 Ingress annotation을 기준으로 TLS Secret 생성 | private key는 민감정보, 인증서는 공개정보 | ASM 이전 비권장; Secret key 이름은 현재 YAML에 직접 선언되지 않음 | `gitops-argocd`; cert-manager/edge 구성 | cert-manager 소유 유지 권장. Rollback은 기존 Certificate/Ingress 참조와 TLS Secret 복원 |

## 추가 확인 결과

- `clusters/dev`에서 AWS access key, API token, bearer token 또는 별도 애플리케이션 private key의 평문 정의는 발견되지 않았다.
- task-api Redis credential과 Redis Chart auth는 서로 다른 파일에 중복 정의되어 있어 회전 시 반드시 같은 배포 단계에서 정합성을 유지해야 한다.
- `task-api-platform`에도 연관 credential key가 존재한다.
  - 비추적 로컬 파일: `/home/harim/task-api-platform/.env`의 `DB_USER`, `DB_PASSWORD`
  - 추적 파일:
    - `/home/harim/task-api-platform/helm/task-api/values.yaml`
    - `/home/harim/task-api-platform/helm/task-api/values.dev.yaml`
    - `/home/harim/task-api-platform/helm/task-api/values.eks.yaml`
  - 관련 key: `postgresdb.auth.password`, `postgres.auth.password`, `secretEnv.DB_PASSWORD`, `secretEnv.REDIS_PASSWORD`
- `task-api-platform` Chart는 현재 `secretEnv` 전체를 `argo-task-api-secret`으로 직접 생성한다. ExternalSecret과 리소스 소유권이 충돌하지 않도록 `existingSecret` 또는 Secret 생성 비활성화 기능을 먼저 추가해야 한다.

## 권장 전환 순서

1. `aws-eks-terraform-lab`: Secrets Manager, IRSA/IAM, External Secrets Operator 책임 범위 확정
2. `gitops-argocd`: SecretStore/ClusterSecretStore와 ExternalSecret 구조 도입
3. Alertmanager webhook 전환
4. Grafana 관리자 credential 전환
5. Redis 서버 auth를 existing Secret으로 전환
6. task-api Chart에 existing Secret 지원 추가
7. task-api DB·Redis credential 전환
8. 모든 소비자 확인 후 Git 평문 key 제거
9. Git history에 노출된 credential 회전
10. cert-manager ACME/TLS key는 별도 요구가 없다면 cert-manager 관리 유지

공통 rollback 조건은 기존 credential을 즉시 폐기하지 않고 새 Secret 동기화와 소비자 정상 동작을 먼저 확인하는 것이다. 회전까지 완료한 뒤에는 이전 값 복원 대신 AWS Secrets Manager의 직전 버전을 재활성화하는 방식이 안전하다.
