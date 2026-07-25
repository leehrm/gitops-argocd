# 운영도구 HTTPS 노출

## 아키텍처

- `grafana.lhrm-lab.com` → Cloudflare DNS → Traefik HTTPS → Grafana 자체 로그인
- `prometheus.lhrm-lab.com` → Cloudflare DNS → Traefik HTTPS + BasicAuth → Prometheus
- `alertmanager.lhrm-lab.com` → Cloudflare DNS → Traefik HTTPS + BasicAuth → Alertmanager
- `argocd.lhrm-lab.com` → Cloudflare DNS → Traefik HTTPS → Argo CD 자체 로그인
- cert-manager는 HTTP-01 Challenge를 Traefik `web` entrypoint로 처리하고 서비스별 TLS Secret을 발급한다.

## Cloudflare DNS

인증서 발급 전에는 모두 **DNS only**, TTL은 Auto로 유지한다.

| Type | Name | Content | Proxy |
|---|---|---|---|
| CNAME | grafana | `a24ce8354efd948bdabdfd18dcfaf446-1011065984.ap-northeast-1.elb.amazonaws.com` | DNS only |
| CNAME | prometheus | 동일 | DNS only |
| CNAME | alertmanager | 동일 | DNS only |
| CNAME | argocd | 동일 | DNS only |

## 재구축 후 체크리스트

ExternalDNS 도입 후에는 DNS도 자동 갱신되므로 수동 작업이 없다. 아래는 확인 절차다.

```bash
kubectl -n external-dns get pods
kubectl -n external-dns logs deploy/external-dns --tail=50
kubectl get ingress -A                       # 7개 ADDRESS가 새 LB hostname으로 일치
dig +short grafana.lhrm-lab.com              # 새 LB hostname으로 갱신되었는지
dig +short TXT edns-grafana.lhrm-lab.com     # owner=eks-dev 유지 확인
kubectl get certificate -A                   # 5장 READY=True
```

Ingress, Middleware, Secret, DNS 모두 Argo CD·ESO·ExternalDNS가 자동 복구한다.

ExternalDNS가 동작하지 않을 때의 수동 fallback:

1. `kubectl -n traefik get svc traefik -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'`
2. Cloudflare CNAME 5개(`task-api` 포함)를 새 LB hostname으로 수정한다.

## staging 인증서 시험

대상 Ingress의 `cert-manager.io/cluster-issuer`를 `letsencrypt-staging`으로 바꾸어 동기화한 뒤 기존 TLS Secret을 삭제해 재발급한다. 시험 후 production issuer로 되돌리고 TLS Secret을 다시 삭제해 production 인증서를 발급한다.

Let's Encrypt 한도는 도메인당 주 50장이다. 재구축 1회에 5장을 쓰므로 주 10회가 상한이며, 초과 시 `too many certificates already issued`가 발생한다.

## 검증

```bash
kubectl -n monitoring get externalsecret
kubectl -n monitoring get secret observability-basic-auth
kubectl get certificate -A
kubectl get ingress -A

for h in grafana prometheus alertmanager argocd; do
  printf "%-14s http:" "$h"
  curl -s -o /dev/null -w "%{http_code}" "http://$h.lhrm-lab.com"
  printf "  https:"
  curl -s -o /dev/null -w "%{http_code}\n" "https://$h.lhrm-lab.com"
done

curl -I http://task-api.lhrm-lab.com
```

기대값은 Grafana `301/302`, Prometheus `301/401`, Alertmanager `301/401`, Argo CD `301/200`이다. task-api HTTP도 계속 `301`이어야 한다.

Argo CD CLI는 Traefik 백엔드가 평문 HTTP/1.1이므로 h2c gRPC 대신 grpc-web을 사용한다.

```bash
argocd login argocd.lhrm-lab.com --grpc-web
```

재구축 후 초기 비밀번호:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```

## 장애 대응

- 인증서 `Ready=False`: Certificate → CertificateRequest → Order → Challenge → solver Ingress → DNS → IngressClass
- 404: Ingress host → DNS → backend Service 이름 → port → EndpointSlice
- 502/503: Service selector → EndpointSlice → Pod readiness → BasicAuth Secret 존재 여부
- Redirect loop: Cloudflare SSL mode → Traefik TLS 종료 → `server.insecure` → forwarded headers
- Cloudflare 525/526: 원본 인증서 → TLS Secret → Certificate Ready → Full (strict)

## 롤백

`root` Application이 `clusters/dev/applications`를 `selfHeal` + `prune`으로 감시하므로 `kubectl delete application <name>`은 약 3분 내 되살아난다. **롤백은 Git revert로 한다.**

```bash
git revert <commit>
git push origin deploy/dev
```

Argo CD가 동기화하면서 리소스를 제거한다. 이후 남는 것을 수동으로 정리한다.

```bash
kubectl -n monitoring delete secret observability-basic-auth
kubectl -n external-dns delete secret cloudflare-api-token
```

`deletionPolicy: Retain`이므로 ExternalSecret이 사라져도 Secret은 남는다. ASM 값은 `AWSPREVIOUS`로 복원할 수 있고 Terraform 컨테이너에는 `prevent_destroy`가 적용되어 있다. task-api와 port-forward 경로는 유지된다.

## ExternalDNS

Ingress의 host를 읽어 Cloudflare에 CNAME을 자동 등록·갱신한다. 재구축으로 LB hostname이 바뀌어도 사람이 DNS를 손대지 않는다.

```text
Ingress(host) → ExternalDNS → Cloudflare API → CNAME + TXT(소유권)
```

| 설정 | 값 | 이유 |
|---|---|---|
| `policy` | `upsert-only` | 삭제를 원천 차단. 재구축 시 필요한 동작은 타깃 갱신(upsert)뿐이다 |
| `registry` | `txt` | 소유권을 TXT로 표시해 남의 레코드를 덮어쓰지 않는다 |
| `txtOwnerId` | `eks-dev` | **변경 금지.** 바뀌면 기존 레코드 소유권을 잃는다 |
| `txtPrefix` | `edns-` | CNAME과 TXT 이름을 분리한다. DNS 규격상 CNAME은 같은 이름에 다른 레코드와 공존할 수 없다 |
| `sources` | `[ingress]` | Service를 넣으면 Traefik LB 자기 자신을 등록한다 |
| `--ingress-class` | `traefik` | 감시 대상을 Traefik Ingress로 한정한다 |

`--cloudflare-proxied` 는 **지정하지 않는다.** 인자 파서(kingpin)가 불리언 플래그의 `=false` 형식을 거부해 `flag parsing error: unexpected false` 로 기동에 실패한다. 기본값이 이미 `false`(DNS only)라 HTTP-01 발급에 문제가 없다. 특정 호스트만 Proxied로 바꾸려면 해당 Ingress에 `external-dns.alpha.kubernetes.io/cloudflare-proxied` annotation을 붙인다.

TTL은 지정하지 않는다. Ingress annotation으로만 설정 가능한데 `task-api` Ingress까지 고쳐야 하고, 재구축 자체가 15분 이상이라 전파 시간(5분→1분) 단축이 묻힌다.

### 최초 인계

기존 5개 레코드는 손으로 만들어 TXT 소유권이 없다. 1회 삭제 후 ExternalDNS가 재생성하게 한다.

1. 배포 후 로그가 `All records are already up to date`인지 확인한다. 기존 레코드에는 소유권 TXT가 없어 ExternalDNS가 손대지 않는 것이 정상이다.
2. 시험 실행이 필요하면 chart 값이 아니라 `extraArgs`에 `--dry-run`을 넣는다. chart 1.21.1에는 `dryRun` 값이 없어 무시된다.
3. `grafana` CNAME 1건만 삭제하고 CNAME + `edns-grafana` TXT가 생성되는지 본다.
4. 성공하면 `prometheus`, `alertmanager`, `argocd`를 삭제한다.
5. 마지막에 `task-api`를 삭제한다.

### 주의

- **호스트명을 바꾸거나 서비스를 제거하면 옛 CNAME과 `edns-` TXT가 남는다.** `upsert-only`는 삭제하지 않으므로 Cloudflare에서 수동으로 지운다.
- 토큰을 회전하면 Pod가 자동 재시작되지 않는다.

```bash
kubectl -n external-dns rollout restart deploy external-dns
```

### 토큰

Cloudflare API 토큰은 `Zone:DNS:Edit` + `Zone:Zone:Read` 권한으로 `lhrm-lab.com` 단일 zone에만 발급한다. Global API Key는 사용하지 않는다.

```text
Cloudflare → ASM /aws-eks-terraform-lab/dev/dns/cloudflare (API_TOKEN)
          → ExternalSecret → Secret cloudflare-api-token (api-token) → CF_API_TOKEN
```
