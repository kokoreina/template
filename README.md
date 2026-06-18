# W10 - Progressive Delivery with Analysis

GitOps setup for API deployment với Argo Rollouts + AnalysisTemplate.

## Concept

Deploy API với **canary strategy** và **automated analysis**:
- Rollout: 10% → 50% → 100%
- AnalysisTemplate query Prometheus để check success rate ≥ 95%
- Auto rollback nếu analysis fail
- AlertManager gửi email khi có SLO violation



## W10 Morning Labs - RBAC + Gatekeeper

Repo này đã được bổ sung đủ Lab 1.1 → 1.3 theo slide `W10 Sáng — Secure & Operate: RBAC + Admission Policy`.

### Lab 1.1 - RBAC

Thư mục mới:

```
rbac/
├── roles.yaml
└── rolebindings.yaml
argocd/apps/rbac.yaml
```

Nghiệm thu:

```bash
kubectl auth can-i create deploy -n demo --as alice        # yes
kubectl auth can-i create deploy -n kube-system --as alice # no
kubectl auth can-i get pods -A --as bob                    # yes
kubectl auth can-i delete nodes --as carol                 # no
```

### Lab 1.2 - Gatekeeper

Thư mục mới:

```
gatekeeper/
├── templates/      # ConstraintTemplate/Rego
├── constraints/    # 4 constraint deny
└── tests/          # manifest để chụp evidence
argocd/apps/gatekeeper.yaml
argocd/apps/gatekeeper-templates.yaml
argocd/apps/gatekeeper-constraints.yaml
```

4 luật enforce:

1. Cấm image tag `:latest`.
2. Bắt buộc `resources.limits.cpu` và `resources.limits.memory`.
3. Cấm `runAsUser: 0`.
4. Cấm `hostNetwork: true`.

### Lab 1.3 - Custom policy

Đã chọn option: reject `Deployment/StatefulSet/Rollout` nếu `replicas > 5`.

Test:

```bash
kubectl apply -f gatekeeper/tests/bad-replicas.yaml # phải reject
```

### Chạy toàn bộ bằng GitOps

Sau khi đổi `repoURL` trong các file `argocd/apps/*.yaml` và `argocd/root.yaml` về repo fork của bạn:

```bash
kubectl apply -f argocd/root.yaml
kubectl get applications -n argocd
```

Kiểm tra evidence Gatekeeper:

```bash
kubectl apply -f gatekeeper/tests/bad-latest.yaml
kubectl apply -f gatekeeper/tests/bad-no-limits.yaml
kubectl apply -f gatekeeper/tests/bad-root.yaml
kubectl apply -f gatekeeper/tests/bad-hostnetwork.yaml
kubectl apply -f gatekeeper/tests/bad-replicas.yaml
kubectl apply -f gatekeeper/tests/good-pod.yaml
kubectl delete -f gatekeeper/tests/good-pod.yaml
```

## Requirements

- Docker Desktop
- kubectl
- minikube
- git

## Structure

```
w10/
├── app-api/              # API Rollout manifests
│   ├── rollout.yaml      # Argo Rollout với canary strategy
│   ├── service.yaml      # Service expose API
│   └── servicemonitor.yaml # Prometheus metrics scraper
├── app-analysis/         # Analysis manifests
│   └── analysis-template.yaml # Template phân tích success rate
├── app-alert/            # Alert manifests
│   ├── prometheus-rules.yaml # PrometheusRule cho SLO alerts
│   ├── email-secret.yaml # Gmail password (NOT COMMITTED)
│   └── README.md         # Alert setup guide
├── app-common/           # Common resources
│   └── demo-namespace.yaml # Namespace demo
├── src/                  # Source code
│   └── api/              # Flask API application
├── argocd/
│   ├── apps/             # ArgoCD Application manifests
│   │   ├── app-api.yaml  # Deploy API Rollout
│   │   ├── app-analysis.yaml # Deploy AnalysisTemplate
│   │   ├── app-alert.yaml # Deploy PrometheusRule
│   │   ├── app-common.yaml # Deploy common resources
│   │   ├── k8s-prometheus.yaml # Prometheus + AlertManager
│   │   └── k8s-rollout.yaml # Argo Rollouts controller
│   └── root.yaml         # App of Apps pattern
└── README.md
```

## Quick Start

### 1. Setup Cluster
```bash
minikube start -p w10 --driver=docker
kubectl config use-context w10
```

### 2. Install ArgoCD
```bash
kubectl create ns argocd
kubectl apply --server-side -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
```

### 3. Access ArgoCD UI
```bash
# Port forward
kubectl -n argocd port-forward svc/argocd-server 8080:443 &

# Get password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

### 4. Deploy App of Apps
```bash
kubectl apply -f argocd/root.yaml
```

### 5. Setup Email Alert (Optional)
```bash
# Follow instructions in app-alert/README.md
cp app-alert/email-secret.yaml.example app-alert/email-secret.yaml
kubectl apply -f app-alert/email-secret.yaml
```

## Components

### Core
- **Argo Rollouts**: Progressive delivery controller
- **Prometheus Stack**: Metrics collection + AlertManager
- **API**: Flask application với metrics endpoint

### GitOps Applications
- `app-api`: API Rollout với canary strategy
- `app-analysis`: AnalysisTemplate cho automated validation
- `app-alert`: PrometheusRule cho runtime alerting
- `app-common`: Shared resources (namespace)
- `k8s-prometheus`: Monitoring stack
- `k8s-rollout`: Argo Rollouts controller

## Verify Deployment

### Check Rollout Status
```bash
# Watch rollout progress
kubectl get rollout api -n demo -w

# Check current state
kubectl get rollout api -n demo

# Check pods
kubectl get pods -n demo -l app=api
```

### Check AnalysisRun
```bash
# List analysis runs
kubectl get analysisrun -n demo

# Watch latest analysis
kubectl get analysisrun -n demo --sort-by=.metadata.creationTimestamp | tail -1

# Describe for detailed metrics
kubectl describe analysisrun -n demo <name>
```

### Query Prometheus Metrics
```bash
# Success rate metric
kubectl run test-query --image=curlimages/curl:latest --rm -i --restart=Never -n monitoring -- \
  curl -s 'http://kube-prometheus-stack-prometheus.monitoring.svc:9090/api/v1/query?query=api:success_rate:5m'
```

## Test Scenarios (GitOps)

### Test 1: Successful Deployment (Success Rate ≥ 90%)
```bash
# Edit rollout to deploy with no errors
nano app-api/rollout.yaml
# Set: ERROR_RATE: "0"

git add app-api/rollout.yaml
git commit -m "test: deploy with 0% error rate"
git push origin main

# Watch AnalysisRun succeed
kubectl get analysisrun -n demo -w
```

### Test 2: Failed Deployment (Success Rate < 90%)
```bash
# Edit rollout to deploy with 15% error rate
nano app-api/rollout.yaml
# Set: ERROR_RATE: "0.15"

git add app-api/rollout.yaml
git commit -m "test: deploy with 15% error rate (should fail)"
git push origin main

# Watch AnalysisRun fail and auto rollback
kubectl get analysisrun -n demo -w
kubectl get rollout api -n demo
```

### Test 3: Trigger SLO Alert Email
```bash
# Edit rollout to set 10% error rate (triggers alert, but passes canary)
nano app-api/rollout.yaml
# Set: ERROR_RATE: "0.10"

git add app-api/rollout.yaml
git commit -m "test: deploy with 10% error rate (90% success)"
git push origin main

# Canary passes (≥90%) but SLO alert fires (below 95%)
# Wait 2-3 minutes, then check email inbox
```


## Configuration Reference

## Lab 2.1 - External Secrets Operator

Added:

```
eso/
argocd/apps/eso.yaml
argocd/apps/eso-config.yaml
```

`argocd/apps/eso.yaml` installs External Secrets Operator first. `argocd/apps/eso-config.yaml` syncs `eso/` after the CRDs exist.

AWS credentials are created manually and must not be committed:

```bash
kubectl create secret generic aws-creds -n demo \
  --from-literal=access-key=REPLACE_WITH_AWS_ACCESS_KEY_ID \
  --from-literal=secret-key=REPLACE_WITH_AWS_SECRET_ACCESS_KEY
```

Evidence:

```bash
kubectl get pods -n external-secrets
kubectl get secretstore -n demo
kubectl describe externalsecret db-secret -n demo
kubectl get secret db-secret -n demo -o jsonpath='{.data.password}' | base64 -d; echo
kubectl get pod secret-reader -n demo -o jsonpath='{.metadata.creationTimestamp}{"\n"}'
kubectl logs secret-reader -n demo --tail=20
git grep -nEi '(AKIA[0-9A-Z]{16}|ASIA[0-9A-Z]{16}|aws_secret_access_key\s*=|BEGIN (ENCRYPTED )?COSIGN PRIVATE KEY)' -- . ':!README.md' ':!eso/README.md' ':!signing/README.md' ':!runbooks/*.md'
```

Local fallback: `eso/secret-store-fake.yaml` uses ESO fake provider for local tests only. The AWS SecretStore remains in Git for the mentor requirement.

## Lab 2.2 - Trivy + Cosign + Admission Verify

Added:

```
signing/
policies/
runbooks/
argocd/apps/policy-controller.yaml
argocd/apps/policies.yaml
```

The build workflow now:

- builds the image from `src/api`
- scans with Trivy and fails on `HIGH` or `CRITICAL`
- pushes to GHCR only after the scan passes
- installs Cosign
- signs the exact semver tag written into `app-api/rollout.yaml`

Cosign private key and password belong in GitHub Secrets:

```bash
cosign generate-key-pair
gh secret set COSIGN_PRIVATE_KEY < cosign.key
gh secret set COSIGN_PASSWORD
cp cosign.pub signing/cosign.pub
```

Also paste the same public key into `policies/cluster-image-policy.yaml`.

Admission evidence:

```bash
kubectl get pods -n cosign-system
kubectl get ns demo --show-labels
kubectl get clusterimagepolicy
kubectl apply -f policies/unsigned-image-test.yaml
cosign verify --key signing/cosign.pub ghcr.io/kokoreina/w10-api:TAG
sed "s/REPLACE_WITH_SIGNED_TAG/TAG/" policies/signed-image-test.yaml | kubectl apply -f -
```

Runbooks:

- `runbooks/eso-rotation-runbook.md`
- `runbooks/supply-chain-runbook.md`
- `runbooks/cve-exception-adr.md`

### Sync Waves
ArgoCD applications deploy in order:
- Wave -2: `external-secrets`, `policy-controller`, `gatekeeper`
- Wave -1: `app-common` (namespace)
- Wave 0: `k8s-prometheus`, `k8s-rollout` (infrastructure)
- Wave -1: `eso-config`, `policies`
- Wave 1: `app-analysis`, `app-alert`, `gatekeeper-constraints` (configuration)
- Wave 2: `app-api` (application)

## Cleanup

```bash
# Delete ArgoCD applications
kubectl delete -f argocd/root.yaml

# Wait for resources to be cleaned up
kubectl get all -n demo
kubectl get all -n monitoring

# Delete ArgoCD
kubectl delete ns argocd

# Stop minikube
minikube stop -p w10
minikube delete -p w10
```
