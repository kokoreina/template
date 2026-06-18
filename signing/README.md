# Lab 2.2 - Cosign key material

Generate a Cosign key pair locally. Commit only `cosign.pub`; never commit `cosign.key`.

```bash
cosign generate-key-pair
```

Store the private key in GitHub Secrets:

```bash
gh secret set COSIGN_PRIVATE_KEY < cosign.key
```

Store the key password in GitHub Secrets:

```bash
gh secret set COSIGN_PASSWORD
```

Replace `signing/cosign.pub` with the generated public key:

```bash
cp cosign.pub signing/cosign.pub
```

Also paste the same public key into `policies/cluster-image-policy.yaml` under `spec.authorities[].key.data`. The public key must match the private key stored in GitHub Secrets, otherwise admission verification will reject the signed image.

Local verification example:

```bash
cosign verify --key signing/cosign.pub ghcr.io/kokoreina/w10-api:TAG
```
