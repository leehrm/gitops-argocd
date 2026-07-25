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

1. `kubectl -n traefik get svc traefik -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'`
2. Cloudflare CNAME 5개(`task-api` 포함)를 새 LB hostname으로 수정한다.
3. `kubectl get certificate -A`에서 전부 `READY=True`인지 확인한다.

Ingress, Middleware, Secret은 Argo CD와 ESO가 자동 복구하므로 DNS만 수동으로 갱신한다.

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

```bash
kubectl -n argocd delete application platform-edge
kubectl -n monitoring delete secret observability-basic-auth
```

`deletionPolicy: Retain`이므로 ExternalSecret 삭제 후 BasicAuth Secret은 수동 삭제해야 한다. ASM 값은 `AWSPREVIOUS`로 복원할 수 있고 Terraform 컨테이너에는 `prevent_destroy`가 적용되어 있다. task-api와 port-forward 경로는 유지된다.

## TODO

ExternalDNS 도입 시 Cloudflare API 토큰 → ASM → ExternalSecret → 컨트롤러 순으로 구성하고, 기존 task-api 레코드를 보호하도록 `policy: upsert-only`를 사용한다.
