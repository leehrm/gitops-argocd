# gitops-argocd

`gitops-argocd`는 ArgoCD 기반 GitOps 배포 상태를 관리하기 위한 repository이다.

이 repository는 `task-api-platform`에서 빌드된 Docker image를 EKS에 배포하기 위한 ArgoCD Application manifest를 관리한다.

---

## Repository 역할

| Repository | 역할 |
|---|---|
| `task-api-platform` | FastAPI 애플리케이션 코드, Dockerfile, Helm Chart, GitHub Actions workflow |
| `task-api-gitops` | FluxCD 기반 GitOps 결과 |
| `gitops-argocd` | ArgoCD 기반 GitOps 배포 상태 |

---

## Branch 전략

```text
main
  - 기본 브랜치
  - README 및 repository 설명
  - 직접 배포 대상 아님

deploy/dev
  - ArgoCD dev-root Application이 바라보는 브랜치
  - dev 환경의 실제 배포 상태 관리
  - GitHub Actions가 image tag 변경 commit을 push하는 브랜치
```

현재 ArgoCD는 `deploy/dev` branch를 기준으로 동작한다.

---

## 파일 역할

| 파일 | 역할 |
|---|---|
| `clusters/dev/bootstrap/root-application.yaml` | ArgoCD가 `gitops-argocd` repository의 `clusters/dev/applications` 경로를 감시하도록 하는 Root Application |
| `clusters/dev/applications/argo-task-api.yaml` | 실제 `task-api`를 배포하는 ArgoCD Child Application |

---

## App of Apps 구조

이 repository는 ArgoCD의 App of Apps 패턴을 사용한다.

```text
dev-root Application
  ↓
gitops-argocd/deploy/dev/clusters/dev/applications 감시
  ↓
argo-task-api Application 생성/갱신
  ↓
task-api-platform Helm Chart 참조
  ↓
EKS argo-task-api namespace에 배포
```

### Root Application

`dev-root` Application은 `gitops-argocd` repository를 감시한다.

```yaml
source:
  repoURL: https://github.com/leehrm/gitops-argocd.git
  targetRevision: deploy/dev
  path: clusters/dev/applications
```

### Child Application

`argo-task-api` Application은 실제 애플리케이션을 배포한다.

```yaml
source:
  repoURL: https://github.com/leehrm/task-api-platform.git
  targetRevision: main
  path: helm/task-api
```

---

## ArgoCD Application 상태 확인

```bash
kubectl get app -n argocd
```

예상 결과:

```text
NAME            SYNC STATUS   HEALTH STATUS
argo-task-api   Synced        Healthy
dev-root        Synced        Healthy
```

---

## 배포 Namespace

실제 애플리케이션은 다음 namespace에 배포된다.

```text
argo-task-api
```

확인:

```bash
kubectl get pods -n argo-task-api
```

예상 결과:

```text
NAME                             READY   STATUS
argo-task-api-xxxxxxxxxx-xxxxx   1/1     Running
argo-task-api-postgresdb-0       1/1     Running
```

---

## Image Tag 업데이트 방식

`task-api-platform` repository의 GitHub Actions workflow가 실행되면 다음 파일의 image tag를 자동으로 수정한다.

```text
clusters/dev/applications/argo-task-api.yaml
```

예시:

```yaml
image:
  repository: 519330023984.dkr.ecr.ap-northeast-1.amazonaws.com/task-api
  tag: "6362dbf"
  pullPolicy: Always
```

GitHub Actions가 이 파일을 수정하고 `deploy/dev` branch에 commit/push하면, ArgoCD가 변경을 감지해 EKS에 새 image를 배포한다.

---

## 현재 배포 이미지 확인

```bash
kubectl get deploy argo-task-api -n argo-task-api \
  -o=jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

예시:

```text
519330023984.dkr.ecr.ap-northeast-1.amazonaws.com/task-api:6362dbf
```

---

## 주요 설정

`argo-task-api.yaml`에서는 Helm values를 override하여 ArgoCD 배포 환경에 맞게 설정한다.

```yaml
image:
  repository: 519330023984.dkr.ecr.ap-northeast-1.amazonaws.com/task-api
  tag: "6362dbf"
  pullPolicy: Always

env:
  DB_HOST: argo-task-api-postgresdb
  DB_PORT: "5432"
  DB_NAME: taskdb
  DB_USER: taskuser
  DB_PASSWORD: taskpass

ingress:
  enabled: false
```

### 설정 이유

| 설정 | 이유 |
|---|---|
| `pullPolicy: Always` | ECR에서 image를 pull하도록 설정 |
| `DB_HOST: argo-task-api-postgresdb` | ArgoCD 배포 시 PostgreSQL Service 이름에 맞춤 |
| `ingress.enabled: false` | Ingress Controller가 없는 상태에서 ArgoCD Health가 Progressing으로 남는 문제 방지 |

---

## 배포 흐름

```text
task-api-platform push
  ↓
GitHub Actions
  ↓
Docker image build
  ↓
ECR push
  ↓
gitops-argocd/deploy/dev image tag 변경
  ↓
dev-root Application이 변경 감지
  ↓
argo-task-api Application 갱신
  ↓
EKS Deployment rollout
```

---

## 정리
- `gitops-argocd`는 ArgoCD가 바라보는 배포 상태 repository이다.
- 애플리케이션 코드는 `task-api-platform`에서 관리하고, 실제 EKS에 어떤 image를 배포할지는 `gitops-argocd`의 `deploy/dev` branch에서 관리한다.
- 이를 통해 CI/CD와 GitOps를 분리하고, 배포 이력을 Git commit으로 추적할 수 있다.