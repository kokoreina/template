# Lab 2.1 - External Secrets Operator

This folder contains the ESO config for namespace `demo`.

- `secret-store.yaml` uses real AWS Secrets Manager in `ap-southeast-1`.
- `secret-store-fake.yaml` is a local fallback for labs without AWS.
- `external-secret.yaml` creates Kubernetes Secret `db-secret` with key `password` and `refreshInterval: 1m`.
- `secret-reader-pod.yaml` mounts `db-secret` as a volume so secret rotation does not require a pod restart.

## AWS credentials

Do this manually in the cluster. Never commit the real access key or secret key:

```bash
kubectl create secret generic aws-creds -n demo \
  --from-literal=access-key=REPLACE_WITH_AWS_ACCESS_KEY_ID \
  --from-literal=secret-key=REPLACE_WITH_AWS_SECRET_ACCESS_KEY
```

The AWS secret in Secrets Manager should exist at key `/demo/db/password`.

## Local fallback without AWS

Use the fake provider only for local evidence when AWS is not available:

```bash
kubectl apply -f eso/namespace.yaml
kubectl apply -f eso/secret-store-fake.yaml
kubectl apply -f eso/external-secret.yaml
kubectl patch externalsecret db-secret -n demo --type merge \
  -p '{"spec":{"secretStoreRef":{"name":"fake-secrets","kind":"SecretStore"},"data":[{"secretKey":"password","remoteRef":{"key":"/demo/db/password","version":"v1"}}]}}'
```

To simulate rotation with fake provider:

```bash
kubectl patch externalsecret db-secret -n demo --type merge \
  -p '{"spec":{"data":[{"secretKey":"password","remoteRef":{"key":"/demo/db/password","version":"v2"}}]}}'
```

## Evidence commands

Check ESO pods:

```bash
kubectl get pods -n external-secrets
kubectl rollout status deploy/external-secrets -n external-secrets
```

Check SecretStore:

```bash
kubectl get secretstore -n demo
kubectl describe secretstore aws-secretsmanager -n demo
kubectl describe secretstore fake-secrets -n demo
```

Check ExternalSecret:

```bash
kubectl get externalsecret -n demo
kubectl describe externalsecret db-secret -n demo
kubectl get secret db-secret -n demo
```

Decode synced secret:

```bash
kubectl get secret db-secret -n demo -o jsonpath='{.data.password}' | base64 -d; echo
```

Prove pod AGE does not change after rotation:

```bash
kubectl apply -f eso/secret-reader-pod.yaml
kubectl get pod secret-reader -n demo -o wide
kubectl get pod secret-reader -n demo -o jsonpath='{.metadata.creationTimestamp}{"\n"}'

# Rotate in AWS Secrets Manager, then wait at least 1 minute.
kubectl get secret db-secret -n demo -o jsonpath='{.data.password}' | base64 -d; echo
kubectl logs secret-reader -n demo --tail=20
kubectl get pod secret-reader -n demo -o jsonpath='{.metadata.creationTimestamp}{"\n"}'
```

Prove no real secret was committed:

```bash
git grep -nEi '(AKIA[0-9A-Z]{16}|ASIA[0-9A-Z]{16}|aws_secret_access_key\s*=|BEGIN (ENCRYPTED )?COSIGN PRIVATE KEY)' -- . ':!README.md' ':!eso/README.md' ':!signing/README.md' ':!runbooks/*.md'
```
