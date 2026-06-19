# Payments Evidence

Use this checklist to capture evidence for the payments onboarding exercise.

## Apply ArgoCD apps

```bash
kubectl apply -f argocd/apps/payments.yaml
kubectl apply -f argocd/apps/payments-app.yaml
```

## Check ArgoCD

```bash
kubectl get applications -n argocd | grep payments
```

PowerShell alternative:

```powershell
kubectl get applications -n argocd | Select-String payments
```

## Check namespace

```bash
kubectl get ns payments --show-labels
```

## Test RBAC

```bash
kubectl auth can-i create deployments -n payments --as payments-dev
kubectl auth can-i create deployments -n demo --as payments-dev
kubectl auth can-i get secrets -n payments --as payments-dev
kubectl auth can-i create rolebindings -n payments --as payments-dev
```

Expected:

```text
create deployments in payments = yes
create deployments in demo = no
get secrets in payments = no
create rolebindings in payments = no
```

## Test ResourceQuota and LimitRange

```bash
kubectl get resourcequota -n payments
kubectl describe resourcequota payments-quota -n payments
kubectl get limitrange -n payments
kubectl describe limitrange payments-default-limits -n payments
```

## Test NetworkPolicy

```bash
kubectl get networkpolicy -n payments
kubectl describe networkpolicy default-deny-ingress -n payments
kubectl describe networkpolicy allow-same-namespace-and-dns-egress -n payments
```

NetworkPolicy requires an enforcing CNI such as Calico. If your cluster has Calico, you can test egress isolation with a temporary curl pod:

```bash
kubectl run payments-curl -n payments --image=curlimages/curl:8.10.1 --restart=Never -it --rm -- \
  sh -c "curl -m 5 http://api.demo.svc.cluster.local || true"
```

Expected: the call to namespace `demo` is blocked or times out when the CNI enforces NetworkPolicy.

## Test Gatekeeper

```bash
kubectl apply -f apps/payments/bad-violation-test.yaml
```

Expected: denied by Gatekeeper because it sets `runAsUser: 0` and omits CPU/memory limits.

## Test app running

```bash
kubectl get deploy,svc,pods -n payments
kubectl rollout status deploy/payments-web -n payments
```
