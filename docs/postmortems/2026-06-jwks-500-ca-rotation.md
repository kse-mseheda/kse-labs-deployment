# Post-mortem: HTTP 500 при fetch JWKS у demo-апах (api-security lab)

**Дата:** 2026-06-15 · **Автор:** mseheda · **Статус:** вирішено (з відомою рекурентністю)
**Зачеплені компоненти:** `spa-token-demo` (і потенційно `grpc/soap/session-cookie-demo`), Keycloak, Vault PKI, ArgoCD

## Короткий зміст
Застосунок повертає **HTTP 500** на кожен автентифікований запит. Причина — **не код і не маніфести**. Застосунок тягне JWKS у Keycloak по HTTPS і не може довіритися TLS-сертифікату Keycloak, бо **живий root CA у Vault був замінений на новий**, а застосунок усе ще довіряє **старому** CA, вшитому в образ. Лікується перепіном поточного CA; корінь — нестабільний Vault PKI.

## Ознаки (як упізнати, що це саме воно)
У логах застосунку:
```
javax.net.ssl.SSLHandshakeException: PKIX path validation failed:
java.security.cert.CertPathValidatorException:
Path does not chain with any of the trust anchors
```
- 500 виникає лише на **автентифікованих** запитах (тих, що тригерять перевірку JWT → fetch JWKS).
- Сам под **Running**, healthy (це відрізняє від іншого збою, див. нижче).

> ⚠️ Якщо под натомість у **`CreateContainerConfigError`** і взагалі не стартує — це **інша** проблема (незасіяний DB-секрет у Vault, ESO `SecretSyncedError`). Її лікування — у issue #4, не тут.

## Корінь проблеми
1. TLS-сертифікат Keycloak випускає cert-manager через Vault PKI: `kse-labs Root CA` → `Intermediate` → лист Keycloak.
2. Vault **перегенерував root з нуля** (новий випадковий ключ → новий fingerprint), напр. `59:77:5B…` → `25:C5:A3…`.
3. Застосунок перевіряє цей TLS **не через JVM-truststore, а через власне джерело довіри** — property `app.internal-ca` (його `JwtConfig`). Туди **вшитий в образ старий CA**.
4. Старий trust anchor не валідує новий ланцюжок → `Path does not chain with any of the trust anchors` → 500.

Тобто це **розбіжність CA** (mismatch), а не протермінований/відкликаний сертифікат.

### Чому root взагалі змінився
root мінтить лише Job `vault-pki-init`, з guard'ом `if pki/ mounted → exit 0`, повішеним як ArgoCD `Sync` hook. Він **тихо перевипускає новий випадковий root щоразу, коли Vault піднімається без `pki/`** — тобто після втрати стану Vault (найімовірніше dev-mode / in-memory: root-токен лежить статично в k8s-секреті). Перемикання на власний форк / re-bootstrap кластера — типовий *привід*, бо піднімає Vault порожнім. Деталі — issue #5.

## Діагностика (3 кроки підтвердження)

**1. Підтвердити, що це trust, а не щось інше** — підробити структурно валідний RS256-JWT і стукнути в захищений ендпоінт:
```sh
APP_HOST="spa-token-demo.192.168.50.10.nip.io"   # підстав свій хост
H=$(printf '{"alg":"RS256","typ":"JWT","kid":"x"}' | base64 | tr '+/' '-_' | tr -d '=')
P=$(printf '{"sub":"x"}' | base64 | tr '+/' '-_' | tr -d '=')
curl -sk -o /dev/null -w '%{http_code}\n' \
  -H "Authorization: Bearer $H.$P.AAAA" \
  "https://$APP_HOST/<protected-endpoint>"
```
- `500` → JWKS TLS trust зламаний (наш випадок).
- `401` → trust ОК (підпис відхилено) — проблема в чомусь іншому.

**2. Звірити fingerprint** живого CA з тим, що вшито/запінено:
```sh
# що зараз віддає Keycloak (issuer ланцюжка):
openssl s_client -connect keycloak.192.168.50.10.nip.io:443 \
  -servername keycloak.192.168.50.10.nip.io -showcerts </dev/null 2>/dev/null \
  | openssl x509 -noout -issuer -fingerprint -sha256
```
Якщо цей fingerprint ≠ тому, що в `app.internal-ca` (ConfigMap `kse-labs-ca` або вшитий в образ) — підтверджено.

## Лікування (GitOps — через git, не live-patch)

**1. Витягнути ПОТОЧНИЙ ланцюжок CA з Vault:**
```sh
TOKEN=$(kubectl -n vault get secret vault-token -o jsonpath='{.data.token}' | base64 -d)
kubectl -n vault exec vault-0 -- sh -c \
  "VAULT_TOKEN='$TOKEN' VAULT_ADDR=http://127.0.0.1:8200 vault read -field=certificate pki/cert/ca"     > root.crt
kubectl -n vault exec vault-0 -- sh -c \
  "VAULT_TOKEN='$TOKEN' VAULT_ADDR=http://127.0.0.1:8200 vault read -field=certificate pki_int/cert/ca" > intermediate.crt
cat root.crt intermediate.crt > internal-ca.pem
openssl x509 -in root.crt -noout -fingerprint -sha256   # звірити, що це новий root
```

**2. Покласти ланцюжок у ConfigMap** `applications/<app>/ca-configmap.yaml` — ключ `internal-ca.pem` (root+intermediate разом).

**3. Перенаправити trust застосунку на цей файл, не чіпаючи образ** — у `deployment.yaml`:
```yaml
env:
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
(Готовий приклад — `applications/spa-token-demo/`, коміт `2711cb3`.)

**4. Закомітити → ArgoCD застосує → рестартнути rollout:**
```sh
kubectl -n applications rollout restart deploy/<app>
```

**5. Перевірити:** повторити крок 1 діагностики — має бути `401` замість `500`.

> ❌ **Що НЕ працює (не повторюй мою першу спробу, коміт `b8cdb3e`):** будувати JVM-truststore через init-контейнер + `-Djavax.net.ssl.trustStore`. Ці застосунки **ігнорують JVM-truststore** — вони дивляться **тільки** в `app.internal-ca`.

## Запобігання (щоб не поверталося)
Перепін CA — це лікування симптому: при наступній втраті стану Vault CA знову перекотиться, і ConfigMap застаріє. Корінь усуває одне з (за надійністю):
1. **Persistent Vault + auto-unseal** (raft/PVC замість dev-mode) — root переживає рестарти.
2. **trust-manager** (cert-manager `Bundle`) — автоматично роздає живий CA-бандл і пересинхронізує після кожної ротації; застосунки лише монтують.
3. Мінімум: у Job `vault-pki-init` **імпортувати фіксований CA** замість `pki/root/generate/internal`.

## Таймлайн
| Час (UTC) | Подія |
|---|---|
| Jun 10 16:47:30 | Vault перевипустив новий root CA (`25:C5:A3…`) |
| Jun 11 10:07 | redeploy застосунку зі старим вшитим CA → 500 |
| Jun 11 14:38 | хибна спроба фіксу (JVM-truststore, `b8cdb3e`) |
| Jun 11 15:07 | робочий фікс (перепін `app.internal-ca`, `2711cb3`) → 401 |

## Посилання
- Issue #5 — причина ротації CA (root cause + remediation)
- Issue #4 — суміжний збій: незасіяний DB-секрет (`CreateContainerConfigError`)
- Коміти: `b8cdb3e` (хибна спроба), `2711cb3` (робочий фікс)
