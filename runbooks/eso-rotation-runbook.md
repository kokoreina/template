# ESO Rotation Runbook

Goal: rotate `/demo/db/password` without restarting the workload.

1. Update the secret value in AWS Secrets Manager at `/demo/db/password`.
2. Wait for ESO to reconcile. This lab uses `refreshInterval: 1m`.
3. Verify Kubernetes Secret changed:

```bash
kubectl get externalsecret db-secret -n demo
kubectl get secret db-secret -n demo -o jsonpath='{.data.password}' | base64 -d; echo
```

4. Verify the reader pod did not restart:

```bash
kubectl get pod secret-reader -n demo
kubectl get pod secret-reader -n demo -o jsonpath='{.metadata.creationTimestamp}{"\n"}'
kubectl logs secret-reader -n demo --tail=20
```

If using the fake provider for local evidence, patch the version from `v1` to `v2` as documented in `eso/README.md`.
