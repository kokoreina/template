# Lab 1.2 + 1.3 - Gatekeeper policies

## Có gì trong folder này?

- `templates/`: 5 ConstraintTemplate bằng Rego.
  - `k8sdisallowedlatesttag`: cấm image `:latest`.
  - `k8srequiredlimits`: bắt buộc `resources.limits.cpu` và `resources.limits.memory`.
  - `k8sdisallowrootuser`: cấm `runAsUser: 0`.
  - `k8sdisallowhostnetwork`: cấm `hostNetwork: true`.
  - `k8smaxreplicas`: Lab 1.3 custom policy, reject replicas > 5.
- `constraints/`: 4 constraint của Lab 1.2 + 1 constraint custom Lab 1.3.
- `tests/`: manifest dùng để chụp evidence pass/reject.

## Thứ tự ArgoCD

1. `argocd/apps/gatekeeper.yaml` cài controller.
2. `argocd/apps/gatekeeper-templates.yaml` sync ConstraintTemplate.
3. `argocd/apps/gatekeeper-constraints.yaml` sync Constraint.

## Lệnh nghiệm thu

```bash
kubectl apply -f gatekeeper/tests/bad-latest.yaml       # phải reject
kubectl apply -f gatekeeper/tests/bad-no-limits.yaml    # phải reject
kubectl apply -f gatekeeper/tests/bad-root.yaml         # phải reject
kubectl apply -f gatekeeper/tests/bad-hostnetwork.yaml  # phải reject
kubectl apply -f gatekeeper/tests/bad-replicas.yaml     # phải reject custom Lab 1.3
kubectl apply -f gatekeeper/tests/good-pod.yaml         # phải pass
kubectl delete -f gatekeeper/tests/good-pod.yaml
```
