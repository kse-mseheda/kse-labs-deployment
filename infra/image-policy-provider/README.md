# image-policy-provider

In-cluster deployment of the [image-policy-provider](https://github.com/kse-bd8338bbe006/image-policy-provider)
service -- an OPA Gatekeeper external-data provider that verifies
container image signatures with Cosign.

## Pieces

| File | What it creates |
| --- | --- |
| `certificate.yaml` | `Certificate` (cert-manager) issued by the in-cluster Vault PKI `ClusterIssuer`. cert-manager writes the keypair + chain into the `image-policy-provider-tls` Secret. |
| `configmap.yaml` | `image-policy-provider-config` ConfigMap holding the verifier's runtime knobs (`ALLOWED_REGISTRIES`, `COSIGN_KEY_PATH`, `COSIGN_IDENTITY`, `COSIGN_OIDC_ISSUER`, `LOG_LEVEL`, `LOG_BODIES`). The Deployment reads it via `envFrom`. Lives in this folder so all provider configuration sits in one place. |
| `deployment.yaml` | The Cosign public-key Secret (lab placeholder), the `Deployment` running the FastAPI service on `:8443` with TLS terminated by uvicorn, and the `Service` exposing port `443`. The Deployment pulls runtime env from `configmap.yaml` via `envFrom`. |
| `cabundle-job.yaml` | RBAC + a `Job` that reads `ca.crt` from the cert-manager Secret and `kubectl apply`s the Gatekeeper `Provider` CRD with the correct `caBundle`. The Provider is not committed as a static manifest because Gatekeeper's webhook rejects an empty `caBundle` and the CA rotates whenever Vault's PKI is re-bootstrapped. |
| `external-secret.yaml` | `ExternalSecret` that materializes the GHCR pull credential from Vault (`secret/github/ghcr-pull-secret`) into a `kubernetes.io/dockerconfigjson` Secret in this namespace. The Deployment references it via `imagePullSecrets`. |

## Sync-wave layout

| wave | what runs |
| --- | --- |
| 19 | `ExternalSecret` (GHCR pull credential) |
| 20 | `Certificate`, `Secret/cosign-key` |
| 25 | `Deployment` + `Service` |
| 30 | RBAC + ConfigMap for the patcher |
| 35 | `Job` that creates the `Provider` with the right caBundle |

## Trust material

The Cosign public key is the only piece that is not committed -- replace
the empty `cosign.pub` in `deployment.yaml` (or supply it via a sealed
secret / external secrets operator) before the verifier can return
`verified` for any image. The verifier returns a clear error string when
no key is configured, which is what we want until students bring their
own.
