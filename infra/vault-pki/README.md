# Vault PKI

One-shot Kubernetes Job that configures HashiCorp Vault as a PKI:

1. Enables `pki/` (root CA, 10-year max TTL) and generates an internal root CA
2. Enables `pki_int/` (intermediate CA, 5-year max TTL) and signs the intermediate with the root
3. Creates the `cluster-services` role used by `cert-manager` to issue certs

The Job is idempotent: it checks whether the engines already exist before
running the setup, so re-applying is safe.

The `cert-manager` `Issuer` (in `infra/cert-manager/`) uses the same Vault
address and the `cluster-services` role to mint TLS certs for in-cluster
services (e.g. the OPA Gatekeeper external-data provider).
