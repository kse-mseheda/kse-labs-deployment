# Gatekeeper Policies

Lab OPA Gatekeeper policies, deployed by ArgoCD as a single Application
(`infra-gatekeeper-policies`). The ApplicationSet picks this directory up
automatically and recurses through subdirectories.

## Layout

One directory per policy. Each directory is a self-contained unit with:

- `template.yaml` -- `ConstraintTemplate` (policy class), vendored
  verbatim from the upstream
  [gatekeeper-library](https://github.com/open-policy-agent/gatekeeper-library)
  so upstream bumps are a clean diff.
- `constraint.yaml` -- our `Constraint` instance. Sets `match` rules,
  `parameters`, and `enforcementAction` (`dryrun` / `warn` / `deny`).

```
infra/gatekeeper-policies/
  README.md
  read-only-root-fs/
    template.yaml
    constraint.yaml
  <next-policy>/
    template.yaml
    constraint.yaml
```

## Current policies

| Directory | Kind | Constraint name | Enforcement |
|---|---|---|---|
| `read-only-root-fs/` | `K8sPSPReadOnlyRootFilesystem` | `pods-must-use-read-only-root-fs` | `dryrun` |
| `required-labels/` | `K8sRequiredLabels` | `deployments-must-have-owner-label` | `deny` |

## Sync ordering

Within a single sync, ArgoCD applies resources in sync-wave order. The
`template.yaml` has no wave (default `0`), the `constraint.yaml` has
wave `1`, so Gatekeeper has a chance to create the generated CRD before
the Constraint references it. If the first sync still races, ArgoCD
retries -- the Application goes `OutOfSync` -> `Progressing` -> `Healthy`
within one or two retry cycles.

## Adding a new policy

1. Create a new subdirectory under `gatekeeper-policies/`.
2. Drop in `template.yaml` (from gatekeeper-library) and `constraint.yaml`.
3. Annotate `constraint.yaml` with `argocd.argoproj.io/sync-wave: "1"`.
4. Start in `enforcementAction: dryrun`.
5. Check audit results in
   `kubectl get <constraint-kind> <name> -o jsonpath='{.status.violations}'`.
6. Fix or exempt violations, then flip to `deny`.
