# Post-mortem: HTTP 500 у authz-demo (lecture 6) — застосунок не монтує CA взагалі

**Дата:** 2026-06-19 · **Автор:** mseheda · **Статус:** вирішено
**Зачеплені компоненти:** `authz-demo` (Keycloak Authorization Services + OpenFGA, lecture 6), Keycloak, ArgoCD

## Короткий зміст
`authz-demo` повертає **HTTP 500** на кожен автентифікований запит (`/kc/documents`, `/kc/documents/{id}`), тоді як `/api/health` живий і под `Running`. Симптом і стектрейс — **ті самі**, що в [post-mortem про ротацію CA](2026-06-jwks-500-ca-rotation.md): застосунок тягне JWKS у Keycloak по HTTPS і не довіряє його TLS-серту. Але **корінь інший**: цього разу `kse-labs-ca` ConfigMap **актуальний** (живий лист Keycloak чисто чейниться до нього), а `authz-demo` **взагалі не монтує** цей CA і не виставляє `app.internal-ca`. Тобто це не дрейф Vault PKI, а **пропущена обв'язка довіри при онбордингу нового апа**. Лікується додаванням тих самих трьох шматків, що вже є в `spa-token-demo`.

## Ознаки (як упізнати, що це саме воно)
У логах застосунку:
```
org.springframework.security.oauth2.jwt.JwtException: An error occurred while attempting to decode the Jwt:
Couldn't retrieve remote JWK set:
  ...I/O error on GET request for
  "https://keycloak.192.168.50.10.nip.io/realms/api-security/protocol/openid-connect/certs":
javax.net.ssl.SSLHandshakeException: PKIX path validation failed:
java.security.cert.CertPathValidatorException:
Path does not chain with any of the trust anchors
```
- 500 лише на **автентифікованих** запитах (ті, що тригерять перевірку JWT → fetch JWKS).
- `/api/health` віддає `{"status":"UP"}`, под **Running**, healthy.
- У deployment **немає** ні `SPRING_APPLICATION_JSON` з `app.internal-ca`, ні volume з `kse-labs-ca` (на відміну від `spa-token-demo`).

## Корінь проблеми
1. Spring resource server при валідації bearer-токена тягне JWKS з `https://keycloak.../realms/api-security/protocol/openid-connect/certs` по HTTPS.
2. Лист Keycloak підписаний `CN=kse-labs Intermediate CA` (Vault PKI). Застосунок перевіряє цей TLS **не через JVM-truststore, а через `app.internal-ca`** (його `JwtConfig`).
3. `authz-demo` **не передає** `app.internal-ca` на жоден актуальний CA-файл і не монтує `kse-labs-ca`. Тож валідатор не має trust anchor для ланцюжка Keycloak → `Path does not chain with any of the trust anchors` → 500.

> 🔑 **Ключова відмінність від [CA-rotation post-mortem](2026-06-jwks-500-ca-rotation.md):** там CA **застарів** (Vault перекотив root, ConfigMap відстав). Тут ConfigMap **актуальний** — перевірено: живий лист Keycloak валідується проти `kse-labs-ca` чисто (`openssl verify ... OK`). Проблема не в *вмісті* CA, а в тому, що апп його **не споживає**. Vault чіпати **не треба**.

### Чому так сталося
`authz-demo` додали для lecture 6 (`deployment.yaml` вперше з'явився 2026-06-17). Обв'язку довіри до CA, яку колись завели у `spa-token-demo` (коміт `2711cb3`), **не пропагували** в маніфест нового апа. Кожен новий demo-апп, що валідує токени Keycloak, потребує цих трьох шматків — але нема ні спільної бази (kustomize/template), ні чеклиста онбордингу, тож їх легко забути. Класичний *скопіювали образ, забули інфра-обв'язку*.

## Діагностика (2 кроки підтвердження)

**1. Підтвердити, що це trust, а не щось інше** — підробити структурно валідний RS256-JWT і стукнути в захищений ендпоінт:
```sh
APP_HOST="authz-demo.192.168.50.10.nip.io"
H=$(printf '{"alg":"RS256","typ":"JWT","kid":"x"}' | base64 | tr '+/' '-_' | tr -d '=')
P=$(printf '{"sub":"x"}' | base64 | tr '+/' '-_' | tr -d '=')
curl -sk -o /dev/null -w '%{http_code}\n' \
  -H "Authorization: Bearer $H.$P.AAAA" \
  "https://$APP_HOST/kc/documents"
```
- `500` → JWKS TLS trust зламаний (наш випадок).
- `401` → trust ОК (підпис відхилено) — проблема в чомусь іншому.

**2. Довести, що ConfigMap АКТУАЛЬНИЙ** (це й відрізняє цей кейс від ротації CA) — звірити живий лист Keycloak проти CA з `kse-labs-ca`:
```sh
# CA, який реально лежить у ConfigMap applications/kse-labs-ca
kubectl -n applications get configmap kse-labs-ca \
  -o go-template='{{range $k,$v := .data}}{{$v}}{{end}}' > /tmp/kse-ca.pem
# живий лист Keycloak
echo | openssl s_client -connect keycloak.192.168.50.10.nip.io:443 \
  -servername keycloak.192.168.50.10.nip.io 2>/dev/null | openssl x509 > /tmp/kc.pem
openssl verify -CAfile /tmp/kse-ca.pem /tmp/kc.pem
```
- `OK` → ConfigMap живий, апп просто його не монтує (наш випадок → фікс нижче, **без** Vault).
- `error 20/21 unable to get local issuer` → CA застарів → це вже **інший** кейс, дивись [CA-rotation post-mortem](2026-06-jwks-500-ca-rotation.md).

## Лікування (GitOps — через git, не live-patch)
> ⚠️ `svc-authz-demo` має ArgoCD `automated` + `selfHeal`. Будь-який `kubectl patch`/`set env` буде **відкочено**. Правити **тільки** в git.

ConfigMap уже актуальний, тож кроки з витягуванням CA з Vault **не потрібні**. Лишається обв'язка в `deployment.yaml` — дзеркало `spa-token-demo`, три шматки:
```yaml
env:
  # Апп валідує TLS JWKS-ендпоінта Keycloak через app.internal-ca, а НЕ JVM-truststore.
  - name: SPRING_APPLICATION_JSON
    value: '{"app":{"internal-ca":"file:/etc/kse-pki/internal-ca.pem"}}'
volumeMounts:
  - name: kse-pki
    mountPath: /etc/kse-pki
    readOnly: true
volumes:
  - name: kse-pki
    configMap:
      name: kse-labs-ca
```
Закомітити → запушити в `main` → ArgoCD синкає сам (бо змінився pod-template, котиться новий ReplicaSet; rollout restart руками не потрібен).

**Перевірити:** повторити крок 1 діагностики — має бути `401` замість `500`; у логах нового пода — 0 рядків `PKIX`/`SSLHandshake`. Реальний токен (`$TOKEN_ALICE`) тепер проходить автентифікацію і віддає 200.

> ❌ **Що НЕ працює:** JVM-truststore через init-контейнер + `-Djavax.net.ssl.trustStore`. Ці апи **ігнорують** JVM-truststore — дивляться **тільки** в `app.internal-ca` (деталі — [CA-rotation post-mortem](2026-06-jwks-500-ca-rotation.md), коміт `b8cdb3e`).

## Запобігання (щоб не поверталося)
Корінь — не Vault, а **онбординг нового апа без інфра-обв'язки**. Усуває одне з (за надійністю):
1. **trust-manager** (cert-manager `Bundle`) — роздає живий CA-бандл у всі ns і пересинхронізує після ротацій; новий апп лише монтує. Закриває і цей кейс, і ротацію CA одним рішенням.
2. **Спільна kustomize-база / template** для demo-апів, де CA-mount + `app.internal-ca` зашиті за замовчуванням — новий апп успадковує, забути неможливо.
3. **Чеклист онбордингу demo-апа**: пункт «змонтувати `kse-labs-ca` + виставити `app.internal-ca`» поряд з «завести DB-секрет».

## Таймлайн
| Час (UTC) | Подія |
|---|---|
| Jun 17 ~03:35 | `authz-demo` онбордять для lecture 6; `deployment.yaml` без CA-mount |
| Jun 19 13:34 | 500 на `GET /kc/documents` (Alice list / get doc 1) |
| Jun 19 13:34–13:49 | діагностика: forged-JWT → 500; `openssl verify` проти `kse-labs-ca` → **OK** (ConfigMap актуальний) |
| Jun 19 ~13:49 | фікс `4b366bd` (CA-mount + `app.internal-ca`) закомічено та запушено в `main` |
| Jun 19 ~13:50 | ArgoCD синкнув, новий под; forged-JWT → **401**, логи чисті |

## Посилання
- [Суміжний post-mortem](2026-06-jwks-500-ca-rotation.md) — **той самий симптом, інший корінь** (ротація CA / дрейф Vault PKI). Спершу відрізни кейси кроком 2 діагностики.
- Зразок обв'язки — `applications/spa-token-demo/deployment.yaml`.
- Коміти: `4b366bd` (фікс authz-demo), `2711cb3` (первинна обв'язка у spa-token-demo).
