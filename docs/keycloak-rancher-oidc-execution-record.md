# Keycloak · Rancher OIDC 설정 실행 기록

- 실행일: 2026-08-08
- 대상 환경: EKS 개발 클러스터
- Keycloak: `26.7.0`
- 기준 Branch: `gitops-argocd/deploy/dev`
- 기준 Commit: `e057d60` (`docs: add keycloak rancher oidc runbook (#59)`)
- 실행 Runbook: [keycloak-rancher-oidc-runbook.md](./keycloak-rancher-oidc-runbook.md)

이 문서는 Admin Console에서 실제로 반영한 설정과 검증 결과를 기록한다. Password, Client Secret,
OTP Seed, Token은 기록하지 않는다.

## 1. 현재 상태

| 항목 | 상태 |
|---|---|
| Keycloak Runtime·DB·TLS | 완료 |
| 영구 관리자 `keycloak-admin` | 생성 및 검증 완료 |
| 영구 관리자 MFA | OTP 등록 및 로그인 검증 완료 |
| `platform` Realm | 생성 완료 |
| Rancher OIDC Client·Mapper | 생성 완료 |
| Rancher Group·Test User | 생성 완료 |
| UserInfo Group Claim | 검증 완료 |
| Test User Password | 미설정 — 사용자 입력 필요 |
| 임시 관리자와 Bootstrap Secret | 아직 존재 — 삭제 승인 대기 |
| Rancher 설치·OIDC 연결 | 후속 단계 |

## 2. 영구 관리자 검증

`master` Realm에서 다음 내용을 확인했다.

- 현재 Admin Console Session: `keycloak-admin`
- User 상태: Enabled
- 직접 부여된 Realm Role: `admin`
- Credential: Password, OTP
- `keycloak-admin` + Password + MFA로 새 로그인 성공

기존 `temp-admin` User와 `keycloak-initial-admin` Kubernetes Secret은 영구 관리자 검증 중에는
삭제하지 않았다.

## 3. `platform` Realm

다음 Realm을 생성했다.

```text
Realm: platform
Issuer: https://keycloak.lhrm-lab.com/realms/platform
```

외부 OIDC discovery에서 다음 Endpoint를 확인했다.

```text
Authorization: https://keycloak.lhrm-lab.com/realms/platform/protocol/openid-connect/auth
Token:         https://keycloak.lhrm-lab.com/realms/platform/protocol/openid-connect/token
UserInfo:      https://keycloak.lhrm-lab.com/realms/platform/protocol/openid-connect/userinfo
```

검증 명령:

```bash
curl --fail --silent --show-error \
  https://keycloak.lhrm-lab.com/realms/platform/.well-known/openid-configuration \
  | jq -r '[.issuer,.authorization_endpoint,.token_endpoint,.userinfo_endpoint]'
```

## 4. Group

`platform` Realm에 다음 최상위 Group을 생성했다.

```text
/rancher-admins
/rancher-operators
/rancher-viewers
```

`/rancher-admins`에만 `realm-management` Client의 다음 최소 조회 역할을 부여했다.

```text
query-users
query-groups
view-users
```

## 5. Rancher OIDC Client

| 항목 | 값 |
|---|---|
| Client type | `OpenID Connect` |
| Client ID | `rancher` |
| Name | `Rancher` |
| Client authentication | `ON` |
| Standard flow | `ON` |
| Direct access grants | `OFF` |
| Valid redirect URI | `https://rancher.lhrm-lab.com/verify-auth` |
| Web origin | `https://rancher.lhrm-lab.com` |

Client Secret은 열거나 복사하지 않았다. Rancher OIDC 연결 시점에 사용자가 직접 Password Manager로
보관하고 Rancher UI에 입력한다.

## 6. Client Mapper

`rancher-dedicated` Client scope에 다음 Mapper를 생성했다.

### Groups Mapper

| 항목 | 값 |
|---|---|
| Type | `Group Membership` |
| Token claim | `groups` |
| Full group path | `OFF` |
| Add to ID token | `OFF` |
| Add to access token | `OFF` |
| Add to userinfo | `ON` |

### Client Audience

| 항목 | 값 |
|---|---|
| Type | `Audience` |
| Included client audience | `rancher` |
| Add to ID token | `OFF` |
| Add to access token | `ON` |

### Group Path

| 항목 | 값 |
|---|---|
| Type | `Group Membership` |
| Token claim | `full_group_path` |
| Full group path | `ON` |
| Add to ID token | `ON` |
| Add to access token | `ON` |
| Add to userinfo | `ON` |

## 7. Test User와 Group 연결

| User | Group | Password |
|---|---|---|
| `kc-admin-test` | `/rancher-admins` | 미설정 |
| `kc-operator-test` | `/rancher-operators` | 미설정 |
| `kc-viewer-test` | `/rancher-viewers` | 미설정 |
| `kc-no-role-test` | 없음 | 미설정 |

## 8. UserInfo Claim 검증

Admin Console의 `Clients > rancher > Client scopes > Evaluate > Generated user info`에서 확인했다.
`sub` 같은 User 고유 식별자는 이 문서에 기록하지 않았다.

| User | `groups` | `full_group_path` |
|---|---|---|
| `kc-admin-test` | `["rancher-admins"]` | `["/rancher-admins"]` |
| `kc-operator-test` | `["rancher-operators"]` | `["/rancher-operators"]` |
| `kc-viewer-test` | `["rancher-viewers"]` | `["/rancher-viewers"]` |
| `kc-no-role-test` | 없음 | 없음 |

Claim은 모두 JSON Array 형식이며 Rancher OIDC 요구사항과 일치한다.

## 9. 비밀정보 처리

이번 작업에서 다음 값을 문서·Git·Tool 출력에 남기지 않았다.

- `keycloak-admin` Password와 OTP Seed
- Test User Password
- `rancher` Client Secret
- Access Token, ID Token, Refresh Token
- Bootstrap Admin Password

`keycloak-initial-admin` Secret은 이름·Type·Data Key만 확인했고 값은 읽지 않았다.

## 10. 남은 작업

1. 사용자가 Test User 4명의 Password를 서로 다르게 설정하고 `Temporary=OFF`로 저장한다.
2. 각 Test User의 실제 로그인을 필요한 단계에서 확인한다.
3. 사용자 승인 후 `master` Realm의 `temp-admin` User를 삭제한다.
4. 같은 승인 범위에서 `keycloak-initial-admin` Kubernetes Secret을 삭제한다.
5. 영구 관리자 MFA 로그인, Bootstrap Secret 부재, `platform` discovery를 다시 확인한다.
6. Realm Export는 Client Secret·Credential이 제거된 sanitized 파일만 별도 Commit한다.

## 11. Keycloak 단계 완료 조건

- Test User Password 설정 완료
- `temp-admin` User와 `keycloak-initial-admin` Secret 제거
- `keycloak-admin` MFA 로그인 유지
- `platform` OIDC discovery 정상
- UserInfo Group Claim 검증 유지
- 비밀정보가 Git에 없음

위 조건이 완료되면 Keycloak 준비 단계를 종료하고 Rancher 설치 단계로 이동한다.
