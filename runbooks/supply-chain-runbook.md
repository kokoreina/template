# Supply Chain Runbook

CI protects the image before it reaches the cluster:

1. GitHub Actions builds the image from `src/api`.
2. Trivy scans the local image and exits with failure for `HIGH` or `CRITICAL` CVEs.
3. If the scan passes, the workflow logs in to GHCR and pushes the image.
4. Cosign signs the same semver tag that is written into `app-api/rollout.yaml`.

Admission protects the cluster:

1. Sigstore Policy Controller runs in `cosign-system`.
2. Namespace `demo` is opted in with `policy.sigstore.dev/include=true`.
3. `ClusterImagePolicy require-signed-w10-api` verifies `ghcr.io/*/w10-api*` signatures with the committed public key.
4. Unsigned images are rejected during admission; signed images are admitted.

Evidence commands:

```bash
kubectl get pods -n cosign-system
kubectl get ns demo --show-labels
kubectl get clusterimagepolicy
kubectl apply -f policies/unsigned-image-test.yaml
cosign verify --key signing/cosign.pub ghcr.io/kokoreina/w10-api:TAG
```
