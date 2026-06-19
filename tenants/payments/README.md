# Payments Tenant

This folder onboards team `payments` as an isolated tenant.

## Why Role and RoleBinding

`payments-dev` gets a namespaced `Role` and `RoleBinding`, so the user can manage basic workloads only inside namespace `payments`. There is no `ClusterRoleBinding`, because that would risk granting permissions outside the tenant boundary.

The Role allows pods, deployments, replicasets, services, and configmaps. It does not grant `secrets`, `roles`, or `rolebindings`, so these checks should stay denied:

```bash
kubectl auth can-i get secrets -n payments --as payments-dev
kubectl auth can-i create rolebindings -n payments --as payments-dev
```

## Why old guardrails apply automatically

The existing Gatekeeper constraints match Pods, Deployments, and other workloads cluster-wide, excluding only infrastructure namespaces. `payments` is not excluded, so the old guardrails apply without writing new policy:

- no `:latest` image tag
- required CPU and memory limits
- no explicit `runAsUser: 0`
- no `hostNetwork: true`
- max replicas is 5

The label `security.kokoreina.dev/guardrails=enabled` documents the intent, but the existing Gatekeeper rules apply because of their match scope.

## NetworkPolicy enforcement

NetworkPolicy requires a CNI that enforces it, such as Calico. If minikube uses the default bridge CNI, the objects may exist but traffic may not actually be blocked.

Recommended minikube start for network evidence:

```bash
minikube start -p w10 --driver=docker --cni=calico
```
