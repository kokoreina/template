# Payments App

`deployment.yaml` and `service.yaml` define a small valid app for team `payments`.

The valid workload includes:

- CPU and memory requests/limits, so the existing Gatekeeper `K8sRequiredLimits` constraint allows it.
- `runAsNonRoot: true` and `runAsUser: 101`, so it does not run as root.
- `allowPrivilegeEscalation: false`.
- `readOnlyRootFilesystem: true`, with writable `emptyDir` mounts for nginx runtime paths.
- No `hostNetwork`.
- A pinned image tag, not `:latest`.

`bad-violation-test.yaml` intentionally violates old policies by omitting limits and setting `runAsUser: 0`. It is used to prove the existing Gatekeeper guardrails also protect the new tenant.

The test manifests include `argocd.argoproj.io/hook: Skip` so ArgoCD does not continuously apply them as part of the healthy application sync. Run them manually for evidence.

Test:

```bash
kubectl apply -f apps/payments/bad-violation-test.yaml
```

Expected: admission is denied by Gatekeeper.
