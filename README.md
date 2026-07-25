# gitops-argocd

`gitops-argocd`는 ArgoCD 기반 GitOps 배포 상태를 관리하는 repository입니다.

이 repository는 `task-api-platform`에서 빌드된 Docker image를 EKS에 배포하기 위한 ArgoCD Application manifest를 관리합니다. 애플리케이션 코드는 `task-api-platform`에서 관리하고, 실제 EKS에 어떤 image tag를 배포할지는 `gitops-argocd`의 `deploy/dev` branch에서 관리합니다.

---

## Repository 역할

| Repository | 역할 |
|---|---|
| `task-api-platform` | FastAPI 애플리케이션 코드, Dockerfile, Helm Chart, GitHub Actions workflow를 관리. |
| `task-api-gitops` | FluxCD 기반 GitOps 학습 결과. |
| `gitops-argocd` | ArgoCD 기반 GitOps 배포 상태를 관리. |

`task-api-gitops`에서는 FluxCD로 GitOps 흐름을 학습했고, `gitops-argocd`에서는 ArgoCD Auto Sync 기반 배포 흐름을 분리하기 위해 repository를 새로 구성했습니다.

---

## Branch 전략

현재 사용 중인 branch는 다음과 같습니다.

```text
main
  - 기본 브랜치입니다.
  - README 및 repository 설명을 관리합니다.
  - ArgoCD가 직접 바라보는 배포 브랜치는 아닙니다.

deploy/dev
  - ArgoCD root Application이 바라보는 브랜치입니다.
  - dev 환경의 실제 배포 상태를 관리합니다.
  - GitHub Actions가 image tag 변경 commit을 push하는 브랜치입니다.
```

현재 ArgoCD는 `deploy/dev` branch를 기준으로 동작합니다.

```text
GitHub Actions가 deploy/dev에 image tag 변경 commit push
  ↓
ArgoCD root Application이 변경 감지
  ↓
argo-task-api Application 갱신
  ↓
EKS에 새 image 배포
```

---

## Directory 구조

```text
gitops-argocd
├── README.md
├── docs
└── clusters
    └── dev
        ├── applications      # Child Application 정의. root가 이 경로를 감시한다
        ├── cert-manager      # ClusterIssuer
        ├── edge              # Ingress, Middleware (task-api, platform)
        ├── karpenter         # NodePool, EC2NodeClass
        ├── monitoring        # ServiceMonitor, AlertmanagerConfig
        └── secretops         # ClusterSecretStore, ExternalSecret
```

---

## 파일 역할

| 파일 | 역할 |
|---|---|
| `clusters/dev/applications/*.yaml` | ArgoCD Child Application. `root`가 이 디렉터리를 감시하며, 각 파일이 Helm chart 또는 이 repository의 다른 경로를 배포한다. |
| `clusters/dev/applications/argo-task-api.yaml` | 실제 `task-api` 애플리케이션을 배포하는 ArgoCD Child Application. |

`root` Application 자체는 이 repository에 없다. `aws-eks-terraform-lab`의 `cluster/argocd.tf`가 `argocd-apps` chart로 생성하며, 감시 대상 repository·branch·경로는 같은 모듈의 `argocd_root_app_*` 변수로 정해진다.

---

## App of Apps 구조

이 repository는 ArgoCD의 App of Apps 패턴을 사용합니다.

```text
root Application
  ↓
gitops-argocd/deploy/dev/clusters/dev/applications 감시
  ↓
argo-task-api Application 생성 또는 갱신
  ↓
task-api-platform repository의 Helm Chart 참조
  ↓
EKS argo-task-api namespace에 배포
```

### Root Application

`root` Application은 `gitops-argocd` repository를 감시합니다. 이 Application은 Terraform(`aws-eks-terraform-lab/cluster/argocd.tf`)이 생성하므로 이 repository에는 정의가 없습니다.

```yaml
source:
  repoURL: https://github.com/leehrm/gitops-argocd.git
  targetRevision: deploy/dev
  path: clusters/dev/applications
```

`targetRevision: deploy/dev`. ArgoCD가 이 branch를 감시하기 때문에 GitHub Actions도 같은 branch에 image tag 변경 commit을 push해야 합니다.

### Child Application

`argo-task-api` Application은 실제 애플리케이션을 배포합니다.

```yaml
source:
  repoURL: https://github.com/leehrm/task-api-platform.git
  targetRevision: main
  path: helm/task-api
```

`argo-task-api` Application은 `task-api-platform`의 Helm Chart를 사용하고, `gitops-argocd`에 정의된 Helm values override를 통해 image tag와 배포 환경 설정을 변경합니다.


---

## Image Tag 업데이트 방식

`task-api-platform` repository의 GitHub Actions workflow가 실행되면 다음 파일의 image tag를 자동으로 수정합니다.

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

GitHub Actions는 다음 순서로 동작합니다.

```text
task-api-platform push
  ↓
GitHub Actions 실행
  ↓
Docker image build
  ↓
Amazon ECR push
  ↓
gitops-argocd/deploy/dev의 image tag 수정
  ↓
gitops-argocd/deploy/dev에 commit/push
  ↓
ArgoCD Auto Sync
  ↓
EKS Deployment rollout
```

---

## Image Tag 전략

이번 구성에서는 Docker image tag로 Git commit SHA 앞 7자리를 사용합니다.

예시:

```text
Git commit SHA: 6362dbf1234...
Docker image tag: 6362dbf
```

Git SHA 기반 tag를 사용하는 이유는 다음과 같습니다.

| 이유 | 설명 |
|---|---|
| 추적성 | 현재 배포된 image가 어떤 Git commit에서 만들어졌는지 확인할 수 있음. |
| Rollback 용이 | 이전 GitOps commit으로 되돌리면 이전 image tag로 배포할 수 있음. |
| `latest`보다 안전 | `latest`는 같은 tag가 계속 덮어써져 어떤 코드인지 추적하기 어려움. |
| GitOps와 적합 | GitOps repo에 image tag 변경 이력이 commit으로 남음. |

수동 테스트에서는 `manual-test` tag를 사용했고, 자동화 이후에는 Git short SHA tag를 사용합니다.

---

## 주요 Helm Values Override

`argo-task-api.yaml`에서는 Helm values를 override하여 ArgoCD 배포 환경에 맞게 설정합니다.

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

설정 이유는 다음과 같습니다.

| 설정 | 이유 |
|---|---|
| `pullPolicy: Always` | EKS Node가 ECR에서 image를 pull하도록 설정합니다. |
| `DB_HOST: argo-task-api-postgresdb` | ArgoCD 배포 시 생성되는 PostgreSQL Service 이름에 맞춥니다. |
| `ingress.enabled: false` | Ingress Controller가 없는 상태에서 ArgoCD Health가 `Progressing`으로 남는 문제를 방지합니다. |

---

## ArgoCD 상태 확인

ArgoCD Application 상태는 다음 명령어로 확인합니다.

```bash
kubectl get app -n argocd
```

결과:

```text
NAME            SYNC STATUS   HEALTH STATUS
argo-task-api   Synced        Healthy
root            Synced        Healthy
```

`root`와 `argo-task-api`가 모두 `Synced / Healthy`이면 GitOps 동기화와 애플리케이션 배포가 정상입니다.

---

## 실제 배포 Image 확인

EKS Deployment에 반영된 image는 다음 명령어로 확인합니다.

```bash
kubectl get deploy argo-task-api -n argo-task-api \
  -o=jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

예시:

```text
519330023984.dkr.ecr.ap-northeast-1.amazonaws.com/task-api:6362dbf
```

이 값이 GitHub Actions가 생성한 Git short SHA tag와 같으면 자동 배포가 정상적으로 완료된 것입니다.

---

## 운영 흐름 요약

현재 구성의 전체 흐름은 다음과 같습니다.

```text
Developer
  ↓
task-api-platform에 code push
  ↓
GitHub Actions
  - Python syntax check
  - Docker build
  - ECR push
  - gitops-argocd image tag update
  ↓
gitops-argocd/deploy/dev
  ↓
ArgoCD root Application
  ↓
ArgoCD argo-task-api Application
  ↓
EKS argo-task-api namespace
```

이 구성에서 GitHub Actions는 Kubernetes에 직접 배포하지 않습니다.
GitHub Actions는 `gitops-argocd`의 image tag만 변경하고, 실제 Kubernetes 배포는 ArgoCD가 수행합니다.