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

## 2. 작업 목적과 단계별 이유

### 영구 관리자와 MFA

Operator가 만든 `temp-admin`은 최초 설정과 복구를 위한 임시 계정이다. 계속 남겨 두면 Password 하나로
전체 Keycloak을 관리하는 불필요한 공격 경로가 된다. 그래서 `admin` Role과 MFA를 가진 영구 관리자를
먼저 검증하고, 잠금 사고를 막기 위해 새 관리자 로그인이 성공한 뒤에만 임시 계정을 삭제한다.

### `platform` Realm 분리

`master` Realm은 Keycloak 자체 관리용이다. Rancher 사용자와 Client를 별도 `platform` Realm에 두면
관리자 계정과 애플리케이션 사용자를 분리할 수 있고, Rancher 설정 오류나 사용자 변경이 `master` 관리
영역에 미치는 범위를 줄일 수 있다.

### Confidential OIDC Client와 Redirect 제한

Rancher Server가 Authorization Code를 Token으로 교환하므로 Client authentication을 사용하는
confidential Client가 필요하다. Redirect URI를 `https://rancher.lhrm-lab.com/verify-auth` 하나로 제한해
인증 결과가 허용하지 않은 주소로 전달되는 것을 막는다. Direct access grants는 필요하지 않아 껐다.

### Group·Mapper·Audience

Keycloak은 사용자를 인증하고 Group 정보를 전달하며, 실제 권한은 Rancher가 부여한다. `groups`와
`full_group_path` Mapper는 Rancher가 UserInfo Endpoint에서 Group을 읽을 수 있게 한다. 두 Claim을
JSON Array로 만든 이유는 Rancher OIDC가 여러 Group을 이 형식으로 처리하기 때문이다. Audience Mapper는
Access Token이 `rancher` Client용임을 명시한다.

### 최소 조회 역할

Rancher에서 사용자와 Group을 찾는 초기 관리자에게만 `query-users`, `query-groups`, `view-users`를
부여했다. `manage-users` 같은 쓰기 권한은 필요하지 않으므로 추가하지 않았다. 조회 역할도
`/rancher-admins`에만 두어 operator와 viewer 권한이 Keycloak 관리 권한으로 확대되지 않게 했다.

### Test User 4명

admin, operator, viewer는 각 Group이 올바른 Claim으로 전달되는 양성 경로를 검증한다. `kc-no-role-test`는
Group이 없는 사용자가 자동으로 권한을 받지 않는지 확인하는 음성 대조군이다. 네 계정을 분리해야 각
권한 경로를 독립적으로 시험하고 과도한 권한 부여를 발견할 수 있다.

### 비밀값 수동 입력과 임시 계정 삭제 순서

Password, Client Secret, OTP Seed는 브라우저나 Git 기록에 남기지 않기 위해 사용자가 직접 입력한다.
Test User Password 설정과 영구 관리자 재로그인까지 끝난 뒤 `temp-admin` User와 Bootstrap Secret을
삭제해 복구 가능성과 보안 정리를 모두 만족시킨다.

## 3. 영구 관리자 검증

`master` Realm에서 다음 내용을 확인했다.

- 현재 Admin Console Session: `keycloak-admin`
- User 상태: Enabled
- 직접 부여된 Realm Role: `admin`
- Credential: Password, OTP
- `keycloak-admin` + Password + MFA로 새 로그인 성공

기존 `temp-admin` User와 `keycloak-initial-admin` Kubernetes Secret은 영구 관리자 검증 중에는
삭제하지 않았다.

## 4. `platform` Realm

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

## 5. Group

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

## 6. Rancher OIDC Client

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

## 7. Client Mapper

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

## 8. Test User와 Group 연결

| User | Group | Password |
|---|---|---|
| `kc-admin-test` | `/rancher-admins` | 미설정 |
| `kc-operator-test` | `/rancher-operators` | 미설정 |
| `kc-viewer-test` | `/rancher-viewers` | 미설정 |
| `kc-no-role-test` | 없음 | 미설정 |

## 9. UserInfo Claim 검증

Admin Console의 `Clients > rancher > Client scopes > Evaluate > Generated user info`에서 확인했다.
`sub` 같은 User 고유 식별자는 이 문서에 기록하지 않았다.

| User | `groups` | `full_group_path` |
|---|---|---|
| `kc-admin-test` | `["rancher-admins"]` | `["/rancher-admins"]` |
| `kc-operator-test` | `["rancher-operators"]` | `["/rancher-operators"]` |
| `kc-viewer-test` | `["rancher-viewers"]` | `["/rancher-viewers"]` |
| `kc-no-role-test` | 없음 | 없음 |

Claim은 모두 JSON Array 형식이며 Rancher OIDC 요구사항과 일치한다.

## 10. 비밀정보 처리

이번 작업에서 다음 값을 문서·Git·Tool 출력에 남기지 않았다.

- `keycloak-admin` Password와 OTP Seed
- Test User Password
- `rancher` Client Secret
- Access Token, ID Token, Refresh Token
- Bootstrap Admin Password

`keycloak-initial-admin` Secret은 이름·Type·Data Key만 확인했고 값은 읽지 않았다.

## 11. 남은 작업

1. 사용자가 Test User 4명의 Password를 서로 다르게 설정하고 `Temporary=OFF`로 저장한다.
2. 각 Test User의 실제 로그인을 필요한 단계에서 확인한다.
3. 사용자 승인 후 `master` Realm의 `temp-admin` User를 삭제한다.
4. 같은 승인 범위에서 `keycloak-initial-admin` Kubernetes Secret을 삭제한다.
5. 영구 관리자 MFA 로그인, Bootstrap Secret 부재, `platform` discovery를 다시 확인한다.
6. Realm Export는 Client Secret·Credential이 제거된 sanitized 파일만 별도 Commit한다.

## 12. Keycloak 단계 완료 조건

- Test User Password 설정 완료
- `temp-admin` User와 `keycloak-initial-admin` Secret 제거
- `keycloak-admin` MFA 로그인 유지
- `platform` OIDC discovery 정상
- UserInfo Group Claim 검증 유지
- 비밀정보가 Git에 없음

위 조건이 완료되면 Keycloak 준비 단계를 종료하고 Rancher 설치 단계로 이동한다.
