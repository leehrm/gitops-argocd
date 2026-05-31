# Week 7 Karpenter Disruption Experiments

## 개요
Karpenter가 노드를 자동으로 생성하고 삭제하는 기본 동작을 확인한 뒤, 다음 disruption 관련 기능들이 실제로 어떻게 동작하는지 검증.

- Disruption Budget
- Consolidation Policy
- PodDisruptionBudget
- `karpenter.sh/do-not-disrupt`
- 실험 후 리소스 정리 기준

> Pod 수요 증가 및 disruption 제어에 따른 Karpenter 동작.  

---

## 환경 정보

| 항목 | 값 |
|---|---|
| Cluster name | `aws-eks-terraform-lab-eks` |
| Region | `ap-northeast-1` |
| Karpenter version | `1.12.1` |
| InstanceProfile | `KarpenterNodeInstanceProfile-aws-eks-terraform-lab-eks` |
| Discovery tag | `karpenter.sh/discovery=aws-eks-terraform-lab-eks` |
| GitOps repo 기준 브랜치 | `deploy/dev` |
| 심화 작업 브랜치 | `feature/week7-karpenter-disruption` |
| Karpenter 리소스 경로 | `clusters/dev/karpenter` |
| 테스트 namespace | `karpenter-test` |

---

## Baseline

실험 시작 전 기준 상태는 다음과 같다.

```text
ArgoCD Applications:
  argo-task-api           Synced / Healthy
  dev-root                Synced / Healthy
  karpenter-autoscaling   Synced / Healthy

EC2NodeClass:
  default Ready=True

NodePool:
  default Ready=True
  Nodes=0

NodeClaim:
  No resources found

Nodes:
  Managed NodeGroup 2대만 존재
  workload=karpenter 노드 없음

karpenter-test:
  테스트 Pod 없음
```

기본 NodePool disruption 설정은 다음과 같다.

```yaml
disruption:
  consolidationPolicy: WhenEmptyOrUnderutilized
  consolidateAfter: 1m
  budgets:
    - nodes: "10%"
```

---

## 기본 Karpenter Scale-out / Scale-down 검증

### 목적

Deployment replica를 수동으로 증가시켜 Pending Pod를 만들고, Karpenter가 NodePool/EC2NodeClass 조건에 맞는 EC2 노드를 자동 생성하는지 확인.

### 실험 흐름

```text
inflate replicas=4
→ 기존 Managed NodeGroup에는 nodeSelector 조건이 맞지 않아 Pod Pending
→ Karpenter가 Pending Pod 감지
→ NodeClaim 생성
→ t3.medium On-Demand EC2 Node 생성
→ Pod Running
→ replicas=0
→ Pod 삭제
→ 빈 Karpenter Node 삭제
→ NodeClaim 삭제
```

### 확인 결과

- `inflate` Pod 4개를 생성하자 Karpenter가 `t3.medium` On-Demand NodeClaim 2개를 생성.
- Pod 4개는 Karpenter가 생성한 노드에 Running 상태로 배치됨.
- `replicas=0`으로 줄이자 Pod가 삭제되었고, consolidation에 의해 NodeClaim과 Karpenter 노드가 자동 삭제되었음.

### 정리

Karpenter는 Pending Pod를 감지해 필요한 EC2 노드를 자동으로 생성하고, workload가 사라지면 consolidation 정책에 따라 빈 노드를 정리할 수 있다.

---

## Experiment 1. Disruption Budget

### 목적

Karpenter가 빈 노드를 삭제하려고 할 때, NodePool disruption budget으로 해당 동작을 제한할 수 있는지 확인.

### 설정

`Empty` reason에 대해 disruption budget을 0으로 설정한다.

```yaml
disruption:
  consolidationPolicy: WhenEmptyOrUnderutilized
  consolidateAfter: 1m
  budgets:
    - nodes: "0"
      reasons:
        - Empty
```

### 기대 결과

```text
replicas=4
→ Karpenter 노드 생성

replicas=0
→ Pod 삭제
→ Karpenter 노드는 빈 노드가 됨
→ 하지만 Empty reason budget이 nodes=0
→ Karpenter가 빈 노드를 삭제하지 못함
```

### 확인 내용

- 빈 노드를 자동 삭제하려는 consolidation 동작이 disruption budget에 의해 제한될 수 있음을 확인.
- 실험 후 budget을 기본값으로 원복하자 NodeClaim과 Karpenter 노드가 삭제되었음.

### 해석

Disruption budget은 Karpenter가 노드를 삭제하거나 교체하는 속도와 범위를 제한하는 안전장치.  

---

## Experiment 2. Consolidation Policy 비교

### 목적

`WhenEmpty`와 `WhenEmptyOrUnderutilized`의 차이를 확인.

| 정책 | 의미 |
|---|---|
| `WhenEmpty` | 완전히 빈 노드만 삭제 |
| `WhenEmptyOrUnderutilized` | 빈 노드뿐 아니라 Pod가 남아 있어도 재배치 가능하면 삭제/통합 가능 |

### 확인한 로그

```text
disrupting node(s)
command="Underutilized ... delete"
decision="delete"
pod-count=1
```

### 결과

- Karpenter가 Pod가 1개 남아 있는 노드를 `Underutilized`로 판단.
- 해당 Pod를 다른 노드로 재배치할 수 있다고 보고 노드 삭제를 결정.
- 이후 대상 Node와 NodeClaim이 삭제되었음.

### 해석

삭제 대상 노드에 Pod가 있었으므로 단순 Empty Node 삭제가 아니라 Underutilized consolidation이었다.

### 정리

`WhenEmptyOrUnderutilized`는 Pod가 남아 있는 노드라도, 해당 Pod를 다른 노드로 옮길 수 있고 비용/노드 수를 줄일 수 있으면 consolidation 대상으로 삼을 수 있다.

---

## Experiment 3. PodDisruptionBudget

### 목적

PodDisruptionBudget이 Karpenter의 voluntary disruption/consolidation을 막을 수 있는지 확인.

### 설정

`pdb-demo` Deployment와 PDB를 생성.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pdb-demo
  namespace: karpenter-test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: pdb-demo
  template:
    metadata:
      labels:
        app: pdb-demo
    spec:
      nodeSelector:
        workload: karpenter
      containers:
        - name: pause
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.7
          resources:
            requests:
              cpu: "700m"
              memory: "256Mi"
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: pdb-demo
  namespace: karpenter-test
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: pdb-demo
```

### 확인한 상태

```text
PDB:
  MIN AVAILABLE = 2
  ALLOWED DISRUPTIONS = 0

Pod:
  pdb-demo Pod 2개가 서로 다른 Karpenter 노드에 Running

NodeClaim:
  2개 유지

NodePool:
  consolidationPolicy = WhenEmptyOrUnderutilized
  consolidateAfter = 1m
```

### 확인된 이벤트

```text
DisruptionBlocked
Pdb prevents pod evictions (PodDisruptionBudget=[karpenter-test/pdb-demo])
```

### 결과

- Karpenter가 저활용 노드 consolidation을 시도할 수 있는 상태였지만, PDB가 Pod eviction을 막음.
- `ALLOWED DISRUPTIONS=0` 상태였기 때문에 Pod 하나라도 evict하면 `minAvailable=2` 조건을 위반.
- 그 결과 Node와 NodeClaim에서 `DisruptionBlocked` 이벤트가 발생.

### 해석

Karpenter가 노드를 줄이려면 해당 노드의 Pod를 evict해야 함.  
하지만 PDB가 eviction을 허용하지 않으면 Karpenter의 voluntary disruption도 진행되지 못함.

### 정리

PDB는 애플리케이션 가용성을 보호하기 위해 Karpenter의 voluntary disruption/consolidation을 막을 수 있음.

---

## Experiment 4. do-not-disrupt

### 목적

`karpenter.sh/do-not-disrupt: "true"` annotation이 붙은 Pod가 Karpenter의 voluntary disruption/consolidation을 막는지 확인.

### 설정

`dnd-demo` Deployment의 Pod template에 다음 annotation을 추가했다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dnd-demo
  namespace: karpenter-test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: dnd-demo
  template:
    metadata:
      labels:
        app: dnd-demo
      annotations:
        karpenter.sh/do-not-disrupt: "true"
    spec:
      nodeSelector:
        workload: karpenter
      containers:
        - name: pause
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.7
          resources:
            requests:
              cpu: "700m"
              memory: "256Mi"
```

### 확인된 이벤트

```text
DisruptionBlocked
Pod has "karpenter.sh/do-not-disrupt" annotation
```

### annotation 제거 후 확인

`do-not-disrupt` annotation을 제거한 뒤에는 Karpenter가 다시 해당 노드를 Underutilized로 판단하고 disruption을 진행.

```text
DisruptionTerminating
Disrupting NodeClaim: Underutilized
Evicted pod: Underutilized
Instance is terminating
Drained
```

### 결과

- annotation이 있을 때는 Karpenter의 voluntary disruption이 차단되었음.
- annotation 제거 후에는 Pod eviction과 NodeClaim/Node termination이 진행됨.

### 해석

`do-not-disrupt`는 특정 workload가 올라간 노드를 Karpenter의 자발적 consolidation 대상에서 제외시키는 보호 장치로 사용할 수 있음.

### 주의

`do-not-disrupt`는 Karpenter의 voluntary disruption을 막는 기능.  
수동 노드 삭제, 강제 EC2 종료, interruption, node repair 등 모든 종료 이벤트를 절대적으로 막는 기능은 아님.

---
