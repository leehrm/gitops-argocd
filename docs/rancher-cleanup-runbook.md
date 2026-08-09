# Rancher cleanup runbook

Rancher 제거는 단순 Git 삭제로 끝나지 않는다. Rancher가 Local Cluster에 만든 CRD, webhook,
RBAC, Fleet 리소스가 남기 때문이다. 아래 절차는 `rancher/rancher-cleanup`을 고정된 소스와
컨테이너 digest로 실행한다.

> **파괴적 작업:** 이 cleanup 버전은 Istio와 Prometheus Operator 리소스도 삭제한다.
> Rancher만 제거할 때는 실행하지 말고, 전체 EKS 클러스터를 destroy하기 직전에만 별도 승인 후 실행한다.

## 고정 버전

- Git commit: `420c3c39aba8dd83e203be6e0a77aa70f9ba6c0d`
- cleanup manifest SHA-256: `2bdc1e72f046a6ae52e18a0f4479a98300871b05333e3ba39e4434d714a9464f`
- verify manifest SHA-256: `f9e022f0122d06c360f2a5e4c931ea2179fb0432d249d81feb7d33b0b71066a0`
- cleanup image: `rancher/rancher-cleanup@sha256:6d86467e88e879e63fa865eb17b645eb8b8a4b0127f3564f208c20b70ee22882`

## 실행

1. 현재 context가 폐기할 클러스터인지 확인하고 Argo CD의 재생성을 중지한다.

   ```bash
   kubectl config current-context
   kubectl get nodes
   kubectl scale statefulset argocd-application-controller -n argocd --replicas=0
   ```

2. 고정된 manifest를 내려받아 checksum을 검증하고 image digest를 고정한다.

   ```bash
   curl -fLo /tmp/rancher-cleanup.yaml \
     https://raw.githubusercontent.com/rancher/rancher-cleanup/420c3c39aba8dd83e203be6e0a77aa70f9ba6c0d/deploy/rancher-cleanup.yaml
   curl -fLo /tmp/rancher-verify.yaml \
     https://raw.githubusercontent.com/rancher/rancher-cleanup/420c3c39aba8dd83e203be6e0a77aa70f9ba6c0d/deploy/verify.yaml

   printf '%s  %s\n' \
     2bdc1e72f046a6ae52e18a0f4479a98300871b05333e3ba39e4434d714a9464f /tmp/rancher-cleanup.yaml \
     f9e022f0122d06c360f2a5e4c931ea2179fb0432d249d81feb7d33b0b71066a0 /tmp/rancher-verify.yaml \
     | shasum -a 256 -c -

   sed -i.bak \
     's#rancher/rancher-cleanup:latest#rancher/rancher-cleanup@sha256:6d86467e88e879e63fa865eb17b645eb8b8a4b0127f3564f208c20b70ee22882#' \
     /tmp/rancher-cleanup.yaml /tmp/rancher-verify.yaml
   ```

3. cleanup과 verify를 차례로 실행한다.

   ```bash
   kubectl apply -f /tmp/rancher-cleanup.yaml
   kubectl wait -n kube-system --for=condition=complete job/cleanup-job --timeout=20m
   kubectl logs -n kube-system job/cleanup-job

   kubectl apply -f /tmp/rancher-verify.yaml
   kubectl wait -n kube-system --for=condition=complete job/verify-job --timeout=10m
   kubectl logs -n kube-system job/verify-job | grep -v 'is deprecated' || true
   ```

   마지막 명령의 출력이 비어 있어야 한다. 출력이 남으면 destroy로 넘어가지 않고 원인을 확인한다.

4. Job의 임시 권한을 제거한 뒤 `pre-destroy-cleanup.sh`와 partial destroy를 진행한다.

   ```bash
   kubectl delete -f /tmp/rancher-verify.yaml --ignore-not-found
   kubectl delete -f /tmp/rancher-cleanup.yaml --ignore-not-found
   ```

공식 소스: <https://github.com/rancher/rancher-cleanup>
