# HPA + Karpenter 테스트

## 목적

> 우리 클러스터는 트래픽 증가에 따라 자동으로 서버(노드)도 늘어난다.
실제 `argo-task-api` Service에 HTTP 요청 부하를 발생시켜 다음 end-to-end 흐름을 확인.

```text
실제 API 트래픽 증가
→ argo-task-api Pod CPU 사용률 증가
→ HPA가 Deployment replica 자동 증가
→ 증가한 Pod가 Karpenter NodePool 조건을 요구
→ 기존 Karpenter 노드의 가용 리소스 부족
→ Karpenter가 추가 NodeClaim/Node 생성
→ 새 API Pod가 새 Karpenter 노드에 스케줄링
```

---

## 준비

### metrics-server 확인

HPA는 CPU/Memory 기반 autoscaling을 위해 `metrics.k8s.io` API가 필요.  
이후 metrics-server를 설치, 이를 통해 HPA가 CPU 사용률을 읽을 수 있는 상태가 준비됨.

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

---

## Helm Chart 수정

values.yaml 안에는 nodeSelector, resources, autoscaling 값이 있지만,
templates/deployment.yaml에서는 그 값을 사용하지 않아, ArgoCD values에 값을 넣어도 실제 Deployment에는 반영 안 됨.
Helm chart의 values.yaml 값이 반영되도록 템플릿 변경.


### Deployment resources/nodeSelector 연결

```yaml
 {{- with .Values.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
```

```yaml
 {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### autoscaling.enabled 조건 처리

HPA가 replica를 관리할 수 있도록 `autoscaling.enabled=true`일 때는 Deployment의 `replicas` 필드를 렌더링하지 않도록 수정.

```yaml
{{- if not .Values.autoscaling.enabled }}
replicas: {{ .Values.replicaCount }}
{{- end }}
```

### HPA template 추가

`templates/hpa.yaml`을 추가.

```yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "task-api.fullname" . }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "task-api.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
{{- end }}
```

---

## GitOps 설정 변경

`gitops-argocd`의 `clusters/dev/applications/argo-task-api.yaml`에 Helm values override를 추가.

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "128Mi"
  limits:
    cpu: "1000m"
    memory: "256Mi"

nodeSelector:
  workload: karpenter

autoscaling:
  enabled: true
  minReplicas: 1
  maxReplicas: 6
  targetCPUUtilizationPercentage: 20
```

설정 의미:

```text
resources.requests.cpu=500m:
  Pod가 여러 개 늘어나면 기존 노드 1대에 모두 들어가기 어려워짐.
  Karpenter NodeClaim 추가 생성을 관찰하기 쉽게 설정.

nodeSelector.workload=karpenter:
  argo-task-api Pod가 Karpenter가 생성한 노드에만 스케줄되도록 함.

autoscaling.targetCPUUtilizationPercentage=20:
  실제 endpoint가 가벼워도 HPA가 반응하기 쉽도록 낮게 설정.
```

또한 ArgoCD가 HPA의 replica 변경을 되돌리지 않도록 `ignoreDifferences`를 추가.

```yaml
ignoreDifferences:
  - group: apps
    kind: Deployment
    name: argo-task-api
    namespace: argo-task-api
    jsonPointers:
      - /spec/replicas
```

---

## ArgoCD 반영 후 초기 상태


```bash
kubectl get deploy,hpa,pods -n argo-task-api -o wide
kubectl get nodeclaim
kubectl get nodes -L workload
```

확인 결과:

```text
HPA:
  argo-task-api
  TARGETS: cpu: 0%/20%
  MINPODS: 1
  MAXPODS: 6
  REPLICAS: 1

API Pod:
  argo-task-api Pod 1개 Running
  Karpenter 노드에 스케줄됨

NodeClaim:
  default-hs6lp

Karpenter Node:
  ip-10-0-102-238.ap-northeast-1.compute.internal
  workload=karpenter
```

---

## 실제 API 트래픽 발생

실제 `argo-task-api` Service에 부하를 주기 위해 load generator Pod를 실행.

```bash
for i in 1 2 3 4 5 6 7 8 9 10; do
  kubectl run -n argo-task-api real-api-load-$i \
    --image=curlimages/curl:8.10.1 \
    --restart=Never \
    -- /bin/sh -c 'while true; do curl -s http://argo-task-api:8000/healthz >/dev/null; done'
done
```

---

## HPA Scale-out 결과

HPA watch 결과에서 CPU 사용률이 target을 초과했고, replicas가 자동 증가했다.

```text
cpu: 25%/20%   REPLICAS 1
cpu: 25%/20%   REPLICAS 2
cpu: 39%/20%   REPLICAS 2
cpu: 28%/20%   REPLICAS 4
```


```text
실제 argo-task-api Service에 요청 증가
→ API Pod CPU 사용률 증가
→ HPA target 20% 초과
→ HPA가 Deployment replicas 1 → 2 → 4로 자동 증가
```

---

## Karpenter NodeClaim 추가 생성 결과

부하 증가 후 새 NodeClaim이 추가로 생성되었다.

초기 NodeClaim:

```text
default-hs6lp
  type: t3.medium
  capacity: on-demand
  zone: ap-northeast-1c
  node: ip-10-0-102-238.ap-northeast-1.compute.internal
  ready: True
```

부하 후 추가 NodeClaim:

```text
default-lhppf
  type: t3.medium
  capacity: on-demand
  zone: ap-northeast-1c
  node: ip-10-0-102-7.ap-northeast-1.compute.internal
  ready: Unknown → True
```

Node 상태 변화:

```text
ip-10-0-102-7.ap-northeast-1.compute.internal
  NotReady → Ready
  workload=karpenter
```

이를 통해 Karpenter가 새 EC2 노드를 생성하고, 해당 노드가 클러스터에 join한 것을 확인했다.

---

## 증가한 API Pod 배치 결과

API Pod는 4개까지 증가.

```text
argo-task-api-685cbcc95d-6w2h5
  node: ip-10-0-102-238.ap-northeast-1.compute.internal

argo-task-api-685cbcc95d-cg27r
  node: ip-10-0-102-238.ap-northeast-1.compute.internal

argo-task-api-685cbcc95d-gdj6b
  node: ip-10-0-102-238.ap-northeast-1.compute.internal

argo-task-api-685cbcc95d-mmm8r
  node: ip-10-0-102-7.ap-northeast-1.compute.internal
```

증가한 replica 중 하나가 새로 생성된 Karpenter 노드에 배치되었음.

---

## Scale-down 확인

부하가 줄어든 뒤 HPA가 replica를 자동으로 줄이는 흐름도 확인되었음.

```text
cpu: 0%/20%   REPLICAS 3
cpu: 0%/20%   REPLICAS 1
```

```text
부하 감소
→ CPU 사용률 하락
→ HPA가 replicas 감소
```

---

## 정리

```text
실제 argo-task-api 엔드포인트 트래픽 증가
→ CPU 사용률 상승
→ HPA target 초과
→ HPA replicas 1 → 2 → 4 증가
→ 증가한 Pod가 Karpenter NodePool 조건 요구
→ 기존 Karpenter 노드의 리소스 부족
→ Karpenter가 추가 NodeClaim 생성
→ 새 t3.medium On-Demand 노드 생성
→ 새 API Pod가 새 Karpenter 노드에 배치
```

```text
실제 API 트래픽 증가로 argo-task-api Pod의 CPU 사용률이 HPA target을 초과했고 HPA가 Deployment replicas를 자동 증가시켰음.
증가한 Pod 수요로 인해 기존 Karpenter 노드의 가용 리소스가 부족해졌고]Karpenter가 추가 NodeClaim과 EC2 노드를 생성했음.
```

---
