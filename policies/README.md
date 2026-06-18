# Lab 2.2 - Admission signature verification

Sigstore Policy Controller enforces only namespaces labeled with:

```bash
policy.sigstore.dev/include=true
```

`namespace-label.yaml` applies that label to namespace `demo`. `cluster-image-policy.yaml` requires matching `ghcr.io/*/w10-api*` images to have a valid Cosign signature from the committed public key.

Before testing, replace the placeholder public key in `signing/cosign.pub` and `policies/cluster-image-policy.yaml` with the real key from:

```bash
cosign generate-key-pair
```

## Evidence commands

Check policy-controller:

```bash
kubectl get pods -n cosign-system
kubectl get validatingwebhookconfiguration | grep policy
```

Check namespace opt-in:

```bash
kubectl get ns demo --show-labels
kubectl get ns demo -o jsonpath='{.metadata.labels.policy\.sigstore\.dev/include}{"\n"}'
```

Check ClusterImagePolicy:

```bash
kubectl get clusterimagepolicy
kubectl describe clusterimagepolicy require-signed-w10-api
```

Unsigned image should be rejected:

```bash
kubectl apply -f policies/unsigned-image-test.yaml
```

Signed image should be admitted after replacing the tag:

```bash
TAG=0.0.1
sed "s/REPLACE_WITH_SIGNED_TAG/${TAG}/" policies/signed-image-test.yaml | kubectl apply -f -
kubectl get pod signed-image-test -n demo
kubectl delete pod signed-image-test -n demo
```

Manual signature verification:

```bash
cosign verify --key signing/cosign.pub ghcr.io/kokoreina/w10-api:TAG
```
