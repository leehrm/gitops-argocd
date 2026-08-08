# Keycloak · Rancher OIDC 초기 설정 Runbook

이 문서는 Keycloak `26.7.0`의 임시 관리자를 교체하고 Rancher `2.14.3`용 OIDC 구성을 만드는 절차다.
Password, Client Secret, OTP Seed, Token은 Git·문서·스크린샷에 남기지 않는다.

## 1. 사전 확인

```bash
kubectl get keycloak keycloak -n keycloak
kubectl get certificate keycloak-lhrm-lab-com-tls -n keycloak
curl --fail --silent --show-error \
  https://keycloak.lhrm-lab.com/realms/master/.well-known/openid-configuration \
  | jq -r .issuer
```

다음 값이 모두 확인되어야 한다.

- Keycloak `Ready=True`
- Certificate `Ready=True`
- Issuer `https://keycloak.lhrm-lab.com/realms/master`

## 2. 임시 관리자를 영구 관리자로 교체

1. Bootstrap username을 확인하고 password는 화면에 출력하지 않고 Clipboard로 복사한다.

   ```bash
   kubectl get secret keycloak-initial-admin -n keycloak \
     -o jsonpath='{.data.username}' | base64 --decode; echo
   kubectl get secret keycloak-initial-admin -n keycloak \
     -o jsonpath='{.data.password}' | base64 --decode | pbcopy
   ```

2. <https://keycloak.lhrm-lab.com/admin/> 에 로그인하고 `master` Realm을 선택한다.
3. `Users`에서 영구 관리자 한 명을 만든다. Password는 Git 밖의 Password Manager에 보관하고
   `Temporary`를 끈다.
4. 영구 관리자의 `Role mapping`에서 `Realm roles`의 `admin`을 부여한다.
5. 영구 관리자의 `Required user actions`에 `Configure OTP`를 추가한다.
6. 로그아웃한 뒤 영구 관리자로 로그인해 TOTP를 등록한다.
7. Private Window에서 영구 관리자와 TOTP로 Admin Console 로그인을 다시 확인한다.
8. 확인이 끝난 뒤에만 `master` Realm에서 임시 관리자 경고가 표시되는 기존 Bootstrap User를 삭제한다.
9. Bootstrap Secret을 삭제하고 Clipboard를 비운다.

   ```bash
   kubectl delete secret keycloak-initial-admin -n keycloak
   pbcopy </dev/null
   ```

영구 관리자 로그인 확인 전에는 임시 사용자나 Secret을 삭제하지 않는다.

## 3. `platform` Realm과 Group 생성

`Create realm`에서 `platform` Realm을 만든 뒤 다음 Group을 추가한다.

```text
/rancher-admins
/rancher-operators
/rancher-viewers
```

Self registration은 켜지 않는다.

## 4. Rancher Client 생성

`platform` Realm의 `Clients`에서 다음 Client를 만든다.

| 항목 | 값 |
|---|---|
| Client type | `OpenID Connect` |
| Client ID | `rancher` |
| Client authentication | `ON` |
| Standard flow | `ON` |
| Direct access grants | `OFF` |
| Valid redirect URIs | `https://rancher.lhrm-lab.com/verify-auth` |
| Web origins | `https://rancher.lhrm-lab.com` |

`Credentials`의 Client Secret은 Password Manager에만 보관한다. PR 8에서 Rancher UI에 직접 입력한다.

## 5. Client Mapper 생성

`Clients > rancher > Client scopes > rancher-dedicated > Mappers`에 아래 세 Mapper를 만든다.
표에 없는 옵션은 기본값을 유지한다.

### Groups Mapper

| 항목 | 값 |
|---|---|
| Mapper type | `Group Membership` |
| Token claim name | `groups` |
| Full group path | `OFF` |
| Add to ID token | `OFF` |
| Add to access token | `OFF` |
| Add to userinfo | `ON` |

### Client Audience

| 항목 | 값 |
|---|---|
| Mapper type | `Audience` |
| Included Client Audience | `rancher` |
| Add to ID token | `OFF` |
| Add to access token | `ON` |

### Group Path

| 항목 | 값 |
|---|---|
| Mapper type | `Group Membership` |
| Token claim name | `full_group_path` |
| Full group path | `ON` |
| Add to ID token | `ON` |
| Add to access token | `ON` |
| Add to userinfo | `ON` |

## 6. Test User 생성

Password는 모두 서로 다르게 만들고 `Temporary`를 끈다. Git이나 Runbook에는 기록하지 않는다.

| User | Group |
|---|---|
| `kc-admin-test` | `/rancher-admins` |
| `kc-operator-test` | `/rancher-operators` |
| `kc-viewer-test` | `/rancher-viewers` |
| `kc-no-role-test` | 없음 |

Rancher에서 사용자·Group을 조회할 초기 관리자 계정이 필요하므로 `/rancher-admins` Group의
`Role mapping`에 `realm-management` Client의 `query-users`, `query-groups`, `view-users`를 부여한다.

## 7. 검증과 Export

1. Realm discovery를 확인한다.

   ```bash
   curl --fail --silent --show-error \
     https://keycloak.lhrm-lab.com/realms/platform/.well-known/openid-configuration \
     | jq -r .issuer
   # https://keycloak.lhrm-lab.com/realms/platform
   ```

2. `rancher-dedicated` Client scope의 `Evaluate`에서 각 Test User를 선택한다.
3. Generated user info에서 `groups`와 `full_group_path`가 JSON Array이고 Group이 정확한지 확인한다.
4. `kc-no-role-test`에는 위 Group이 없어야 한다.
5. Admin Console의 partial export는 완전한 Database Backup이 아니다. Export를 Git에 추가할 때는
   Client Secret, 사용자 Credential, Token이 없는지 검토한 sanitized 파일만 별도 Commit한다.

## 8. 완료 조건

- 영구 관리자 + TOTP 로그인 성공
- 임시 관리자 User와 `keycloak-initial-admin` Secret 제거
- `platform` Realm, `rancher` Client, Group 3개, Test User 4개 생성
- `groups`와 `full_group_path` UserInfo Claim 확인
- Secret·Password·OTP 정보가 Git과 문서에 없음

Rancher 설치 후 `Keycloak (OIDC)` Provider를 사용한다. Keycloak 17+에서는 Rancher가 생성한 `/auth`
경로를 쓰지 않고 Endpoint를 `Specify`로 설정해 `platform` Realm의 실제 Issuer를 입력한다.

## 참고

- <https://ranchermanager.docs.rancher.com/v2.14/how-to-guides/new-user-guides/authentication-permissions-and-global-configuration/authentication-config/configure-keycloak-oidc>
- <https://www.keycloak.org/server/bootstrap-admin-recovery>
- <https://www.keycloak.org/docs/latest/server_admin/>
