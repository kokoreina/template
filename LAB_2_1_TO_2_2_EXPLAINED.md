# Lab 2.1 - 2.2 Explained: ESO + Trivy + Cosign + Admission Verify

File này giải thích chi tiết từng file trong Lab 2.1 và Lab 2.2 của repo hiện tại, kèm workflow từng luồng để bạn hiểu vì sao làm như vậy, chạy thế nào, kiểm chứng ra sao.

Nội dung chính:

- Lab 2.1: External Secrets Operator, AWS Secrets Manager, fake provider local, rotate secret không restart pod.
- Lab 2.2: Trivy scan CVE, Cosign ký image, Sigstore Policy Controller verify chữ ký ở admission.

---

# Tổng Quan Lab 2

Lab 2 chia thành 2 trục bảo mật:

```text
Lab 2.1 - Secrets:
Secret không nằm trong Git.
Secret thật nằm ở provider bên ngoài như AWS Secrets Manager.
External Secrets Operator sync secret về Kubernetes Secret.
Pod đọc Secret qua volume để rotate không restart.

Lab 2.2 - Supply Chain:
Image phải được scan CVE trước khi push.
Image phải được ký bằng Cosign.
Cluster chỉ cho chạy image đã ký bằng public key tin cậy.
```

Luồng tổng quát:

```text
GitHub Actions
  -> build image
  -> Trivy scan
  -> push GHCR
  -> Cosign sign
  -> update rollout tag
  -> GitOps deploy
  -> Admission verify image signature
```

---

# Lab 2.1 - External Secrets Operator

Mục tiêu:

```text
Đổi DB password ở nguồn secret bên ngoài.
ESO sync về Kubernetes Secret db-secret trong <= 60s.
Pod đọc secret mới từ mounted volume.
Pod không restart.
Không commit AWS credential thật vào Git.
```

Trong repo có thư mục:

```text
eso/
├── namespace.yaml
├── secret-store.yaml
├── secret-store-fake.yaml
├── external-secret.yaml
├── secret-reader-pod.yaml
└── README.md

argocd/apps/
├── eso.yaml
└── eso-config.yaml
```

## ESO Là Gì?

External Secrets Operator là controller chạy trong cluster. Nó làm nhiệm vụ:

```text
1. Kết nối tới external secret provider, ví dụ AWS Secrets Manager
2. Đọc secret từ provider
3. Tạo hoặc update Kubernetes Secret
4. Lặp lại theo refreshInterval
```

Nó giải quyết vấn đề:

- Không lưu secret thật trong Git.
- Secret có thể rotate từ provider bên ngoài.
- Kubernetes Secret được sync tự động.
- Pod mount Secret bằng volume có thể thấy nội dung mới mà không cần restart.

## File: `argocd/apps/eso.yaml`

File này cài External Secrets Operator bằng Helm chart.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: external-secrets
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
```

Ý nghĩa:

```text
ArgoCD tạo Application tên external-secrets.
Application này cài ESO controller vào namespace external-secrets.
```

Phần source:

```yaml
source:
  repoURL: https://charts.external-secrets.io
  chart: external-secrets
  targetRevision: 0.16.2
```

Giải thích:

- Đây là Helm chart repo chính của External Secrets.
- `chart: external-secrets` là chart cần cài.
- `targetRevision: 0.16.2` là version chart.

Phần Helm values:

```yaml
installCRDs: true
```

Ý nghĩa:

```text
Cài CRD của ESO, ví dụ SecretStore và ExternalSecret.
Nếu CRD chưa có mà apply SecretStore/ExternalSecret thì Kubernetes báo no matches for kind.
```

Phần resources:

```yaml
resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

Ý nghĩa:

- Đặt request/limit cho ESO controller.
- Tránh bị Gatekeeper policy `K8sRequiredLimits` chặn nếu chart resource rơi vào namespace bị enforce.
- Giúp workload có ngân sách tài nguyên rõ ràng.

Phần destination:

```yaml
destination:
  server: https://kubernetes.default.svc
  namespace: external-secrets
```

Ý nghĩa:

```text
ESO controller chạy trong namespace external-secrets.
```

Sync options:

```yaml
CreateNamespace=true
ServerSideApply=true
```

Ý nghĩa:

- `CreateNamespace=true`: ArgoCD tự tạo namespace `external-secrets` nếu chưa có.
- `ServerSideApply=true`: apply resource bằng server-side apply.

Vì sao sync wave `-2`:

```text
ESO operator và CRD phải có trước SecretStore/ExternalSecret.
```

## File: `argocd/apps/eso-config.yaml`

File này sync thư mục `eso/`.

```yaml
metadata:
  name: eso-config
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
```

Ý nghĩa:

```text
eso-config chạy sau external-secrets.
```

Phần source:

```yaml
source:
  repoURL: https://github.com/kokoreina/template.git
  path: eso
  targetRevision: main
```

Ý nghĩa:

```text
ArgoCD đọc toàn bộ manifest trong thư mục eso/ từ branch main.
```

Sync option quan trọng:

```yaml
SkipDryRunOnMissingResource=true
```

Vì sao cần:

- `SecretStore` và `ExternalSecret` là CRD do ESO tạo ra.
- Trong lúc ArgoCD sync, CRD có thể vừa mới được cài.
- Option này giảm lỗi dry-run nếu ArgoCD chưa nhìn thấy CRD ngay lập tức.

## File: `eso/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
  labels:
    name: demo
```

Ý nghĩa:

```text
Đảm bảo namespace demo tồn tại để chứa SecretStore, ExternalSecret, Secret và pod test.
```

Trong repo cũng có `app-common/demo-namespace.yaml`. File này vẫn hữu ích khi chạy riêng Lab 2.1 hoặc khi sync `eso/` độc lập.

## File: `eso/secret-store.yaml`

Đây là SecretStore AWS thật.

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: aws-secretsmanager
  namespace: demo
```

SecretStore trả lời câu hỏi:

```text
ExternalSecret sẽ đọc secret từ provider nào, region nào, authenticate bằng gì?
```

Phần provider:

```yaml
provider:
  aws:
    service: SecretsManager
    region: ap-southeast-1
```

Ý nghĩa:

- Provider là AWS.
- Service là AWS Secrets Manager.
- Region mặc định là `ap-southeast-1`.

Phần auth:

```yaml
auth:
  secretRef:
    accessKeyIDSecretRef:
      name: aws-creds
      key: access-key
    secretAccessKeySecretRef:
      name: aws-creds
      key: secret-key
```

Ý nghĩa:

```text
ESO lấy AWS access key từ Kubernetes Secret tên aws-creds trong namespace demo.
```

Secret `aws-creds` không được commit vào Git. Nó được tạo thủ công:

```bash
kubectl create secret generic aws-creds -n demo \
  --from-literal=access-key=REPLACE_WITH_AWS_ACCESS_KEY_ID \
  --from-literal=secret-key=REPLACE_WITH_AWS_SECRET_ACCESS_KEY
```

Vì sao làm thủ công:

- Không lộ credential trong Git.
- Đúng yêu cầu mentor: "Không commit AWS credentials thật".
- Git chỉ chứa cách tham chiếu secret, không chứa giá trị secret.

## File: `eso/secret-store-fake.yaml`

Đây là SecretStore fake để test local khi chưa có AWS.

```yaml
kind: SecretStore
metadata:
  name: fake-secrets
  namespace: demo
spec:
  provider:
    fake:
      data:
      - key: /demo/db/password
        value: local-fake-password-v1
        version: v1
      - key: /demo/db/password
        value: local-fake-password-v2
        version: v2
```

Ý nghĩa:

```text
ESO fake provider giả lập external provider.
Nó trả về value tĩnh theo key và version.
```

Vì sao cần fake provider:

- Không phải ai cũng có AWS account/credential lúc làm lab.
- Vẫn chứng minh được pattern ESO:
  - ExternalSecret đọc provider.
  - Tạo Kubernetes Secret.
  - Đổi version từ v1 sang v2 để mô phỏng rotation.
  - Pod đọc secret mới mà không restart.

Điểm cần nhớ:

```text
Fake provider chỉ dùng local/evidence.
AWS SecretStore vẫn là file đúng yêu cầu production/mentor.
```

## File: `eso/external-secret.yaml`

Đây là object nói rõ secret nào cần sync về Kubernetes.

Repo hiện tại đang trỏ fake provider:

```yaml
secretStoreRef:
  name: fake-secrets
  kind: SecretStore
```

Ý nghĩa:

```text
ExternalSecret db-secret sẽ đọc từ SecretStore fake-secrets.
```

Nếu muốn chạy AWS thật, đổi về:

```yaml
secretStoreRef:
  name: aws-secretsmanager
  kind: SecretStore
```

Phần refresh:

```yaml
refreshInterval: 1m
```

Ý nghĩa:

```text
ESO reconcile mỗi 1 phút.
Mentor yêu cầu cập nhật <= 60s, nên 1m là phù hợp.
```

Phần target:

```yaml
target:
  name: db-secret
  creationPolicy: Owner
```

Ý nghĩa:

- ESO tạo Kubernetes Secret tên `db-secret`.
- `creationPolicy: Owner` nghĩa là ESO quản lý lifecycle của Secret này.

Phần data:

```yaml
data:
- secretKey: password
  remoteRef:
    key: /demo/db/password
    version: v1
```

Ý nghĩa:

```text
Đọc key /demo/db/password từ provider.
Lưu vào Kubernetes Secret db-secret với key password.
```

Kết quả Kubernetes Secret sẽ có dạng logic:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: demo
data:
  password: <base64-value>
```

## File: `eso/secret-reader-pod.yaml`

Pod này chứng minh secret rotate không cần restart.

```yaml
kind: Pod
metadata:
  name: secret-reader
  namespace: demo
```

Container:

```yaml
image: busybox:1.36
command:
- sh
- -c
- |
  while true; do
    echo "$(date -Iseconds) password=$(cat /mnt/db/password 2>/dev/null || true)"
    sleep 10
  done
```

Ý nghĩa:

```text
Cứ 10 giây pod đọc file /mnt/db/password và in ra log.
```

Secret được mount bằng volume:

```yaml
volumes:
- name: db-secret
  secret:
    secretName: db-secret
```

Mount vào container:

```yaml
volumeMounts:
- name: db-secret
  mountPath: /mnt/db
  readOnly: true
```

Kết quả:

```text
Kubernetes Secret key password trở thành file /mnt/db/password.
```

Vì sao dùng volume, không dùng env:

- Nếu secret được inject bằng env var, app chỉ đọc lúc process start.
- Khi secret đổi, env var trong process không đổi.
- Muốn env var đổi thường phải restart pod.
- Với Secret volume, kubelet tự cập nhật mounted file sau một khoảng thời gian.
- Pod không cần restart; app chỉ cần đọc lại file.

Resources:

```yaml
resources:
  limits:
    cpu: 50m
    memory: 64Mi
```

Ý nghĩa:

```text
Có limits để không bị Gatekeeper Lab 1.2 chặn.
```

## File: `eso/README.md`

Đây là hướng dẫn chạy/evidence cho Lab 2.1.

Nội dung chính:

- Tạo AWS credential thủ công bằng `kubectl create secret`.
- Dùng fake provider nếu chưa có AWS.
- Check ESO pod.
- Check SecretStore.
- Check ExternalSecret.
- Decode secret.
- Chứng minh pod AGE/creationTimestamp không đổi sau rotate.
- Grep repo để chứng minh không có secret thật.

## Workflow Lab 2.1 - Luồng AWS Thật

```text
1. ArgoCD sync argocd/apps/eso.yaml
2. Helm cài ESO controller + CRDs
3. Tạo namespace demo
4. Tạo Kubernetes Secret aws-creds thủ công, không commit Git
5. Tạo secret /demo/db/password trong AWS Secrets Manager
6. ArgoCD sync argocd/apps/eso-config.yaml
7. ESO đọc SecretStore aws-secretsmanager
8. ESO đọc ExternalSecret db-secret
9. ESO gọi AWS Secrets Manager lấy /demo/db/password
10. ESO tạo/update Kubernetes Secret db-secret
11. Pod secret-reader mount db-secret thành file /mnt/db/password
12. Rotate secret trong AWS
13. Sau refreshInterval 1m, ESO update db-secret
14. Mounted file trong pod đổi, pod không restart
```

Lệnh evidence:

```bash
kubectl apply -f argocd/apps/eso.yaml
kubectl get pods -n external-secrets
kubectl rollout status deploy/external-secrets -n external-secrets

kubectl apply -f eso/namespace.yaml
kubectl create secret generic aws-creds -n demo \
  --from-literal=access-key=REPLACE_WITH_AWS_ACCESS_KEY_ID \
  --from-literal=secret-key=REPLACE_WITH_AWS_SECRET_ACCESS_KEY

kubectl apply -f argocd/apps/eso-config.yaml
kubectl get secretstore -n demo
kubectl get externalsecret -n demo
kubectl get secret db-secret -n demo
kubectl get secret db-secret -n demo -o jsonpath='{.data.password}' | base64 -d; echo
kubectl get pod secret-reader -n demo -o jsonpath='{.metadata.creationTimestamp}{"\n"}'
kubectl logs secret-reader -n demo --tail=20
```

Kỳ vọng:

```text
ESO pod Running.
SecretStore Ready.
ExternalSecret Ready/Synced.
Kubernetes Secret db-secret có key password.
Sau rotate, password đổi.
Pod creationTimestamp/AGE không đổi.
```

## Workflow Lab 2.1 - Luồng Fake Provider Local

Repo hiện tại đang thuận tiện cho fake provider vì `external-secret.yaml` trỏ `fake-secrets`.

Luồng:

```text
1. Cài ESO operator bằng argocd/apps/eso.yaml
2. Apply eso/namespace.yaml
3. Apply eso/secret-store-fake.yaml
4. Apply eso/external-secret.yaml
5. ESO tạo db-secret từ fake value v1
6. Apply secret-reader-pod.yaml
7. Pod đọc local-fake-password-v1
8. Patch ExternalSecret version v2
9. ESO update db-secret thành local-fake-password-v2
10. Pod log thấy password mới nhưng pod không restart
```

Lệnh:

```bash
kubectl apply -f argocd/apps/eso.yaml
kubectl rollout status deploy/external-secrets -n external-secrets

kubectl apply -f eso/namespace.yaml
kubectl apply -f eso/secret-store-fake.yaml
kubectl apply -f eso/external-secret.yaml
kubectl apply -f eso/secret-reader-pod.yaml

kubectl get externalsecret db-secret -n demo
kubectl get secret db-secret -n demo -o jsonpath='{.data.password}' | base64 -d; echo
kubectl logs secret-reader -n demo --tail=10
kubectl get pod secret-reader -n demo -o jsonpath='{.metadata.creationTimestamp}{"\n"}'
```

Rotate fake v1 sang v2:

```bash
kubectl patch externalsecret db-secret -n demo --type merge \
  -p '{"spec":{"data":[{"secretKey":"password","remoteRef":{"key":"/demo/db/password","version":"v2"}}]}}'

sleep 70

kubectl get secret db-secret -n demo -o jsonpath='{.data.password}' | base64 -d; echo
kubectl logs secret-reader -n demo --tail=20
kubectl get pod secret-reader -n demo -o jsonpath='{.metadata.creationTimestamp}{"\n"}'
```

Kỳ vọng:

```text
Secret đổi từ local-fake-password-v1 sang local-fake-password-v2.
Pod log đọc value mới.
Pod AGE/creationTimestamp không đổi.
```

## Vì Sao Secret Không Restart Pod?

Kubernetes Secret volume được kubelet sync vào pod filesystem. Khi Secret object đổi, kubelet cập nhật file trong mounted volume sau một khoảng thời gian.

Điều kiện để app thấy secret mới:

```text
App phải đọc lại file từ disk.
Nếu app cache password trong memory mãi mãi, app vẫn cần logic reload.
```

Trong lab, pod dùng loop `cat /mnt/db/password` mỗi 10 giây nên nhìn thấy file mới.

---

# Lab 2.2 - Trivy + Cosign + Admission Verify

Mục tiêu:

```text
CI fail nếu image có CVE HIGH/CRITICAL.
Image chỉ được push sau khi scan pass.
Image được ký bằng Cosign.
Cluster chỉ cho chạy image đã ký.
Unsigned image bị admission reject.
```

Các thư mục/file chính:

```text
.github/workflows/build-push.yml
.github/workflows/validate.yml
signing/cosign.pub
signing/README.md
policies/namespace-label.yaml
policies/cluster-image-policy.yaml
policies/unsigned-image-test.yaml
policies/signed-image-test.yaml
policies/README.md
argocd/apps/policy-controller.yaml
argocd/apps/policies.yaml
runbooks/supply-chain-runbook.md
runbooks/cve-exception-adr.md
```

## File: `.github/workflows/build-push.yml`

Workflow này xử lý supply chain của API image.

Trigger:

```yaml
on:
  push:
    branches:
      - main
    paths:
      - 'src/api/**'
      - '.github/workflows/build-push.yml'
  workflow_dispatch:
```

Ý nghĩa:

- Chạy khi push vào `main` và thay đổi source API hoặc workflow.
- Có thể chạy tay bằng `workflow_dispatch`.

Env:

```yaml
REGISTRY: ghcr.io
IMAGE_NAME: ${{ github.repository_owner }}/w10-api
```

Image sẽ có dạng:

```text
ghcr.io/<owner>/w10-api:<tag>
```

Permissions:

```yaml
contents: write
packages: write
security-events: write
```

Ý nghĩa:

- `contents: write`: workflow commit update `app-api/rollout.yaml` và tạo git tag.
- `packages: write`: push image lên GHCR.
- `security-events: write`: phục vụ security scanning/reporting nếu cần.

### Step: Checkout

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0
```

Ý nghĩa:

```text
Checkout repo và lấy đầy đủ git history để semantic version tính version/tag.
```

### Step: Calculate semantic version

```yaml
uses: paulhatch/semantic-version@v5.4.0
```

Ý nghĩa:

```text
Tính version dạng major.minor.patch từ commit history.
```

Output dùng về sau:

```text
steps.semver.outputs.version
```

Ví dụ:

```text
0.0.2
```

### Step: Extract metadata

```yaml
uses: docker/metadata-action@v5
tags:
  type=raw,value=latest
  type=raw,value=${{ steps.semver.outputs.version }}
  type=sha,prefix=v${{ steps.semver.outputs.version }}-
```

Ý nghĩa:

Workflow tạo nhiều tag:

```text
latest
0.0.x
v0.0.x-<sha>
```

Lưu ý bảo mật:

- Lab yêu cầu ký đúng tag đang deploy.
- Step sign bên dưới ký tag semver, không ký mỗi `latest`.

### Step: Build Docker image for scan and push

```yaml
uses: docker/build-push-action@v6
with:
  context: ./src/api
  load: true
  push: false
  tags: ${{ steps.meta.outputs.tags }}
```

Ý nghĩa:

```text
Build image từ thư mục src/api.
load: true đưa image vào local Docker engine của GitHub runner.
push: false vì chưa được push trước khi scan pass.
```

Vì sao scan trước push:

```text
Nếu image có CVE HIGH/CRITICAL, pipeline fail và image không được publish.
```

### Step: Scan image with Trivy

```yaml
uses: aquasecurity/trivy-action@master
with:
  image-ref: ghcr.io/.../w10-api:${{ steps.semver.outputs.version }}
  severity: HIGH,CRITICAL
  vuln-type: os,library
  ignore-unfixed: true
  exit-code: '1'
```

Ý nghĩa:

- Scan OS packages và library dependencies.
- Chỉ quan tâm HIGH/CRITICAL.
- `exit-code: '1'` làm workflow fail nếu phát hiện CVE matching rule.
- `ignore-unfixed: true` bỏ qua CVE chưa có bản vá để giảm false blocking, nhưng vẫn nên có ADR nếu exception.

Kỳ vọng Lab 2.2:

```text
Push image chứa CVE HIGH/CRITICAL -> CI đỏ.
```

### Step: Login GHCR

```yaml
uses: docker/login-action@v3
with:
  registry: ghcr.io
  username: ${{ github.actor }}
  password: ${{ secrets.GITHUB_TOKEN }}
```

Ý nghĩa:

```text
Đăng nhập GitHub Container Registry để push image.
```

### Step: Push Docker image

```bash
while IFS= read -r tag; do
  if [ -n "$tag" ]; then
    docker push "$tag"
  fi
done <<< "$IMAGE_TAGS"
```

Ý nghĩa:

```text
Push tất cả tag do docker/metadata-action tạo.
```

Chỉ chạy sau khi Trivy scan pass.

### Step: Install Cosign

```yaml
uses: sigstore/cosign-installer@v3.7.0
```

Ý nghĩa:

```text
Cài cosign CLI trên GitHub runner.
```

### Step: Sign deployed image tag

```yaml
env:
  COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
  COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
  IMAGE_TO_SIGN: ghcr.io/.../w10-api:${{ steps.semver.outputs.version }}
run: |
  cosign sign --yes --key env://COSIGN_PRIVATE_KEY "${IMAGE_TO_SIGN}"
```

Ý nghĩa:

```text
Cosign ký image tag semver chính là tag sẽ được ghi vào rollout.yaml.
Private key nằm trong GitHub Secrets, không commit vào Git.
Password cũng nằm trong GitHub Secrets.
```

Vì sao ký đúng tag đang deploy:

- Nếu chỉ ký `latest` nhưng Rollout deploy `0.0.x`, admission verify tag `0.0.x` sẽ fail.
- Lab yêu cầu "sign đúng image tag đang deploy".

### Step: Update rollout.yaml

```bash
sed -i "s|image: ghcr.io/.*/w10-api:.*|image: ghcr.io/.../w10-api:${VERSION}|g" app-api/rollout.yaml
sed -i "s|value: \"v.*\"|value: \"v${VERSION}\"|g" app-api/rollout.yaml
```

Ý nghĩa:

```text
Sau khi image đã scan, push, sign, workflow cập nhật app-api/rollout.yaml sang version mới.
```

ArgoCD sau đó sẽ deploy image mới từ Git.

### Step: Commit and push version update

Workflow commit file `app-api/rollout.yaml` nếu có thay đổi.

Ý nghĩa:

```text
Git vẫn là source of truth.
Cluster không tự deploy image ngoài Git.
```

### Step: Create git tag

```bash
git tag v${VERSION}
git push origin v${VERSION}
```

Ý nghĩa:

```text
Đánh dấu source code tương ứng với image version.
```

## File: `.github/workflows/validate.yml`

Workflow validate manifest trong pull request.

Trigger paths:

```yaml
app-api/**
app-analysis/**
app-alert/**
app-common/**
argocd/**
rbac/**
gatekeeper/**
eso/**
policies/**
signing/**
runbooks/**
```

Ý nghĩa:

```text
Nếu PR thay đổi manifest liên quan Lab 1/2, chạy validate.
```

Step chính:

```bash
kubeconform -strict -summary -ignore-missing-schemas \
  app-api/ app-analysis/ app-alert/ app-common/ argocd/ rbac/ gatekeeper/ eso/ policies/
```

Ý nghĩa:

- `kubeconform` validate YAML/Kubernetes schema.
- `-ignore-missing-schemas` bỏ qua CRD schema chưa có sẵn như ArgoCD Application, Rollout, ExternalSecret, ClusterImagePolicy.

## File: `signing/cosign.pub`

Đây là public key dùng để verify chữ ký image.

```text
-----BEGIN PUBLIC KEY-----
...
-----END PUBLIC KEY-----
```

Ý nghĩa:

```text
Public key được commit vào Git.
Policy Controller dùng public key này để verify image signature.
```

Quan trọng:

- Public key được commit là OK.
- Private key `cosign.key` tuyệt đối không commit.
- Private key phải nằm trong GitHub Secret `COSIGN_PRIVATE_KEY`.
- Password nằm trong GitHub Secret `COSIGN_PASSWORD`.
- Public key trong `signing/cosign.pub` phải khớp private key dùng để ký trong CI.

## File: `signing/README.md`

Hướng dẫn tạo key:

```bash
cosign generate-key-pair
```

Sau đó:

```bash
gh secret set COSIGN_PRIVATE_KEY < cosign.key
gh secret set COSIGN_PASSWORD
cp cosign.pub signing/cosign.pub
```

Ý nghĩa:

```text
Private key đi vào GitHub Secrets.
Public key đi vào Git.
```

Lệnh verify local:

```bash
cosign verify --key signing/cosign.pub ghcr.io/kokoreina/w10-api:TAG
```

## File: `.gitignore`

Phần liên quan Lab 2.2:

```text
cosign.key
*.key
```

Ý nghĩa:

```text
Tránh commit private key Cosign hoặc key file khác.
```

## File: `argocd/apps/policy-controller.yaml`

Cài Sigstore Policy Controller bằng Helm chart.

```yaml
metadata:
  name: policy-controller
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
```

Source:

```yaml
repoURL: https://sigstore.github.io/helm-charts
chart: policy-controller
targetRevision: 0.10.4
```

Destination:

```yaml
namespace: cosign-system
```

Ý nghĩa:

```text
Policy Controller chạy trong namespace cosign-system.
Nó tạo admission webhook để verify image signature khi Pod/workload được tạo.
```

Vì sao wave `-2`:

```text
Controller và CRD phải có trước khi apply ClusterImagePolicy.
```

## File: `argocd/apps/policies.yaml`

Sync thư mục `policies/`.

```yaml
metadata:
  name: policies
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
source:
  repoURL: https://github.com/kokoreina/template.git
  path: policies
```

Ý nghĩa:

```text
Sau khi policy-controller chạy, ArgoCD apply namespace label và ClusterImagePolicy.
```

Option:

```yaml
SkipDryRunOnMissingResource=true
```

Vì sao cần:

- `ClusterImagePolicy` là CRD do Policy Controller cung cấp.
- ArgoCD dry-run có thể chưa thấy CRD ngay sau khi cài chart.

## File: `policies/namespace-label.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
  labels:
    policy.sigstore.dev/include: "true"
```

Ý nghĩa:

```text
Opt-in namespace demo vào Sigstore Policy Controller enforcement.
```

Điểm quan trọng:

- Policy Controller thường chỉ enforce namespace có label `policy.sigstore.dev/include=true`.
- Nếu namespace không có label này, image có thể không bị verify.
- Mentor nhấn mạnh bẫy này: policy chỉ enforce trên namespace được label.

Lưu ý thực tế:

```text
Nên đảm bảo image app đang chạy đã được ký trước khi label namespace.
Nếu label quá sớm, app chưa ký có thể bị admission reject.
```

## File: `policies/cluster-image-policy.yaml`

Policy verify chữ ký image.

```yaml
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: require-signed-w10-api
```

Phạm vi image:

```yaml
images:
- glob: "ghcr.io/*/w10-api*"
```

Ý nghĩa:

```text
Policy áp dụng cho các image w10-api trong GHCR.
```

Authority:

```yaml
authorities:
- name: cosign-public-key
  key:
    data: |
      -----BEGIN PUBLIC KEY-----
      ...
      -----END PUBLIC KEY-----
```

Ý nghĩa:

```text
Chỉ image có chữ ký verify được bằng public key này mới được admission cho chạy.
```

Điểm cần nhớ:

- Public key trong policy phải khớp private key CI dùng để ký.
- Nếu public key sai, image dù có ký cũng bị reject.
- Nếu image chưa ký, bị reject.

## File: `policies/unsigned-image-test.yaml`

Pod test negative:

```yaml
kind: Pod
metadata:
  name: unsigned-image-test
  namespace: demo
spec:
  containers:
  - image: ghcr.io/kokoreina/w10-api:unsigned-test
```

Ý nghĩa:

```text
Thử deploy image không có chữ ký hợp lệ.
```

Kỳ vọng:

```text
Admission reject bởi Sigstore Policy Controller.
```

File có resources limits:

```yaml
resources:
  limits:
    cpu: 50m
    memory: 64Mi
```

Vì sao:

```text
Tránh bị Gatekeeper Lab 1.2 chặn vì thiếu limits.
Lab 2.2 muốn chứng minh bị reject do chữ ký, không phải do policy khác.
```

## File: `policies/signed-image-test.yaml`

Pod test positive:

```yaml
kind: Pod
metadata:
  name: signed-image-test
  namespace: demo
spec:
  containers:
  - image: ghcr.io/kokoreina/w10-api@sha256:c0e7...
```

Ý nghĩa:

```text
Thử deploy image theo digest cụ thể đã được ký/verify.
```

Vì sao dùng digest:

- Digest immutable hơn tag.
- Admission verify kiểm tra đúng artifact.
- Tránh tag bị trỏ sang image khác.

Kỳ vọng:

```text
Nếu digest đã được ký bằng private key tương ứng public key trong policy, pod được admission pass.
Nếu chưa ký hoặc key không khớp, vẫn bị reject.
```

File cũng có resources limits để không bị Gatekeeper chặn.

## File: `policies/README.md`

Hướng dẫn evidence:

- Check policy-controller pod.
- Check namespace `demo` có label opt-in.
- Check `ClusterImagePolicy`.
- Apply unsigned image, kỳ vọng reject.
- Apply signed image, kỳ vọng pass.
- Verify image bằng Cosign CLI.

Lưu ý: README vẫn có đoạn template `REPLACE_WITH_SIGNED_TAG`, còn file hiện tại đã dùng digest cụ thể. Khi test thực tế, hãy dùng image tag/digest đã được CI ký.

## File: `runbooks/supply-chain-runbook.md`

Runbook giải thích ngắn luồng supply chain.

CI protect:

```text
1. Build image từ src/api
2. Trivy scan HIGH/CRITICAL
3. Scan pass thì push GHCR
4. Cosign ký tag semver được deploy
```

Admission protect:

```text
1. Policy Controller chạy trong cosign-system
2. Namespace demo opt-in bằng label
3. ClusterImagePolicy verify image w10-api bằng public key
4. Unsigned image reject, signed image pass
```

## File: `runbooks/cve-exception-adr.md`

Đây là ADR xử lý exception CVE có thời hạn.

Vì sao cần:

```text
Trivy fail pipeline khi có HIGH/CRITICAL.
Nhưng đôi khi vendor chưa có fix hoặc CVE không reachable.
Không nên disable pipeline tùy tiện.
Phải có exception có thời hạn và owner rõ ràng.
```

Thông tin exception cần có:

- CVE ID.
- Package affected.
- Severity.
- Link output Trivy.
- Lý do chấp nhận risk.
- Compensating control.
- Owner.
- Expiry date.
- Fix plan.

Quy trình:

```text
1. Mở PR với exception note.
2. Set expiry 7-14 ngày.
3. Nếu cần, thêm CVE vào .trivyignore trong thời gian approved.
4. Remove exception khi có fix.
```

Quan trọng:

```text
CVE exception không bypass admission signature.
Image vẫn phải được Cosign sign và Policy Controller verify.
```

## File: `runbooks/eso-rotation-runbook.md`

Runbook này thuộc Lab 2.1 nhưng nằm trong nhóm runbook Lab 2.

Nó nhắc lại quy trình:

```text
1. Update secret value ở AWS Secrets Manager /demo/db/password
2. Chờ refreshInterval 1m
3. Verify Kubernetes Secret đổi
4. Verify pod không restart bằng creationTimestamp/log
5. Nếu fake provider thì patch version v1 sang v2
```

---

# Workflow Lab 2.2 - CI Supply Chain

Luồng CI đầy đủ:

```text
1. Developer push code vào main hoặc chạy workflow_dispatch
2. GitHub Actions checkout repo
3. Tính semantic version
4. Tạo Docker tags
5. Build image từ src/api
6. Trivy scan image local
7. Nếu HIGH/CRITICAL CVE -> exit 1 -> workflow fail -> không push
8. Nếu scan pass -> login GHCR
9. Push image lên GHCR
10. Cài Cosign
11. Ký image tag semver bằng COSIGN_PRIVATE_KEY
12. Update app-api/rollout.yaml sang image tag mới
13. Commit và push thay đổi rollout.yaml
14. Tạo git tag v<version>
15. ArgoCD thấy Git thay đổi và sync deployment
```

Kỳ vọng:

```text
Image lỗi CVE HIGH/CRITICAL -> CI đỏ.
Image sạch -> push + ký + update Git.
```

## Workflow Lab 2.2 - Admission Verify

Luồng cluster:

```text
1. ArgoCD sync policy-controller.yaml
2. Helm cài Sigstore Policy Controller vào cosign-system
3. ArgoCD sync policies.yaml
4. policies/namespace-label.yaml label namespace demo
5. policies/cluster-image-policy.yaml tạo ClusterImagePolicy
6. User hoặc ArgoCD tạo Pod/Deployment dùng ghcr.io/*/w10-api*
7. Kubernetes API server gọi Policy Controller admission webhook
8. Policy Controller kiểm tra image có chữ ký hợp lệ không
9. Verify bằng public key trong ClusterImagePolicy
10. Nếu verify fail -> reject
11. Nếu verify pass -> allow
```

Lệnh evidence:

```bash
kubectl get pods -n cosign-system
kubectl get ns demo --show-labels
kubectl get clusterimagepolicy
kubectl describe clusterimagepolicy require-signed-w10-api
kubectl apply -f policies/unsigned-image-test.yaml
kubectl apply -f policies/signed-image-test.yaml
```

Kỳ vọng:

```text
unsigned-image-test = reject
signed-image-test   = pass nếu image digest/tag đã được ký đúng key
```

Manual verify:

```bash
cosign verify --key signing/cosign.pub ghcr.io/kokoreina/w10-api:TAG
```

Hoặc với digest:

```bash
cosign verify --key signing/cosign.pub ghcr.io/kokoreina/w10-api@sha256:<digest>
```

---

# Ba Kiểm Chứng Chính Của Lab 2.2

## 1. Push image chứa CVE HIGH

Kỳ vọng:

```text
GitHub Actions fail ở step Trivy.
Image không được push.
Cosign không ký.
```

## 2. Deploy image chưa ký

Kỳ vọng:

```text
Admission reject bởi Policy Controller.
```

Điều kiện:

- Namespace `demo` có label `policy.sigstore.dev/include=true`.
- `ClusterImagePolicy` đã tồn tại.
- Image match glob `ghcr.io/*/w10-api*`.

## 3. Deploy image đã ký từ CI

Kỳ vọng:

```text
Admission pass.
Pod được tạo.
```

Điều kiện:

- Image đã được Cosign sign.
- Public key trong `ClusterImagePolicy` khớp private key trong GitHub Secrets.
- Đang deploy đúng tag/digest đã ký.

---

# Các Bẫy Thường Gặp

## Bẫy 1: Ký `latest` nhưng deploy tag khác

Nếu CI ký:

```text
ghcr.io/owner/w10-api:latest
```

Nhưng Rollout deploy:

```text
ghcr.io/owner/w10-api:0.0.3
```

Thì admission verify tag `0.0.3` có thể fail.

Repo xử lý bằng cách:

```text
Cosign ký IMAGE_TO_SIGN = semver tag.
Workflow update rollout.yaml sang cùng semver tag.
```

## Bẫy 2: Public key không khớp private key

Nếu `COSIGN_PRIVATE_KEY` trong GitHub Secret không phải cặp với public key trong:

```text
signing/cosign.pub
policies/cluster-image-policy.yaml
```

Thì image đã ký vẫn bị reject.

## Bẫy 3: Namespace chưa opt-in

Nếu namespace `demo` không có:

```text
policy.sigstore.dev/include=true
```

Policy Controller có thể không enforce.

## Bẫy 4: Gatekeeper chặn trước Policy Controller

Nếu test pod thiếu `resources.limits`, Gatekeeper Lab 1.2 sẽ reject trước. Khi đó bạn không chứng minh được lỗi signature.

Vì vậy `unsigned-image-test.yaml` và `signed-image-test.yaml` đều khai báo limits.

## Bẫy 5: Commit private key hoặc AWS key

Không bao giờ commit:

```text
cosign.key
AWS access key
AWS secret access key
```

Repo có `.gitignore` và grep evidence để giảm rủi ro.

---

# Lệnh Nghiệm Thu Tổng Hợp Lab 2.1

Check ESO:

```bash
kubectl get pods -n external-secrets
kubectl rollout status deploy/external-secrets -n external-secrets
```

Check SecretStore/ExternalSecret:

```bash
kubectl get secretstore -n demo
kubectl describe secretstore fake-secrets -n demo
kubectl describe secretstore aws-secretsmanager -n demo
kubectl get externalsecret db-secret -n demo
kubectl describe externalsecret db-secret -n demo
```

Decode secret:

```bash
kubectl get secret db-secret -n demo -o jsonpath='{.data.password}' | base64 -d; echo
```

Check pod no restart:

```bash
kubectl get pod secret-reader -n demo -o wide
kubectl get pod secret-reader -n demo -o jsonpath='{.metadata.creationTimestamp}{"\n"}'
kubectl logs secret-reader -n demo --tail=20
```

Secret scan:

```bash
git grep -nEi '(AKIA[0-9A-Z]{16}|ASIA[0-9A-Z]{16}|aws_secret_access_key\s*=|BEGIN (ENCRYPTED )?COSIGN PRIVATE KEY)' -- . ':!README.md' ':!eso/README.md' ':!signing/README.md' ':!runbooks/*.md'
```

Kỳ vọng:

```text
Không có output nghĩa là không match secret thật theo pattern.
```

---

# Lệnh Nghiệm Thu Tổng Hợp Lab 2.2

Check Policy Controller:

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

Unsigned image:

```bash
kubectl apply -f policies/unsigned-image-test.yaml
```

Kỳ vọng:

```text
Rejected by admission.
```

Signed image:

```bash
kubectl apply -f policies/signed-image-test.yaml
kubectl get pod signed-image-test -n demo
kubectl delete pod signed-image-test -n demo
```

Kỳ vọng:

```text
Pass nếu image digest đã được ký đúng key.
```

Verify local:

```bash
cosign verify --key signing/cosign.pub ghcr.io/kokoreina/w10-api:TAG
```

---

# Tư Duy Kết Hợp Lab 1 Và Lab 2

Lab 1 bảo vệ manifest và quyền truy cập:

```text
RBAC: ai được làm gì
Gatekeeper: manifest có an toàn không
```

Lab 2 bảo vệ secret và supply chain:

```text
ESO: secret không nằm trong Git, rotate tự động
Trivy: image có CVE nặng thì không được phát hành
Cosign: image phát hành phải có chữ ký
Policy Controller: cluster reject image chưa ký
```

Khi kết hợp:

```text
Developer không có quyền quá mức.
Manifest xấu bị reject.
Secret thật không nằm trong Git.
Image lỗi CVE bị chặn từ CI.
Image chưa ký bị chặn ở admission.
```

Đây là chuỗi phòng thủ theo nhiều lớp:

```text
Git -> CI -> Registry -> GitOps -> Admission -> Runtime
```
