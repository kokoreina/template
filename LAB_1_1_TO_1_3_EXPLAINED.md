# Lab 1.1 - 1.3 Explained: RBAC + Gatekeeper + GitOps Workflow

File này giải thích chi tiết các file thuộc Lab 1.1, 1.2, 1.3 trong repo hiện tại, kèm workflow từng luồng từ GitOps đến test bằng `kubectl`.

## Tổng Quan Luồng GitOps

Repo dùng mô hình ArgoCD App of Apps:

```text
argocd/root.yaml
└── sync argocd/apps/
    ├── app-common.yaml
    ├── rbac.yaml
    ├── gatekeeper.yaml
    ├── gatekeeper-templates.yaml
    └── gatekeeper-constraints.yaml
```

Luồng tổng quát:

```text
1. Bạn commit manifest vào Git
2. ArgoCD root app đọc thư mục argocd/apps/
3. Mỗi file trong argocd/apps/ tạo một child Application
4. Child Application sync thư mục tương ứng trong repo
5. Kubernetes API server nhận resource
6. Nếu là workload, Gatekeeper admission webhook kiểm tra policy trước khi cho tạo
```

Các sync wave quan trọng:

```text
Wave -2: gatekeeper controller
Wave -1: app-common namespace demo, gatekeeper templates
Wave  1: rbac, gatekeeper constraints
```

Vì sao cần đúng thứ tự:

- Namespace `demo` phải có trước khi tạo Role/RoleBinding hoặc workload trong namespace đó.
- Gatekeeper controller phải chạy trước khi apply `ConstraintTemplate`.
- `ConstraintTemplate` phải có trước khi apply `Constraint`, nếu không sẽ lỗi kiểu `no matches for kind`.
- `Constraint` phải có trước khi test manifest vi phạm.

---

# Lab 1.1 - RBAC

Mục tiêu: phân quyền 3 user giả lập qua Kubernetes RBAC:

```text
alice = developer trong namespace demo
bob   = SRE thao tác pod toàn cluster
carol = viewer chỉ đọc toàn cluster
```

Điểm cần hiểu: Kubernetes RBAC trả lời câu hỏi "user này có được làm hành động X trên resource Y trong namespace Z không?". Lab dùng `kubectl auth can-i ... --as <user>` để giả lập user, chưa cần authentication thật.

## File: `rbac/roles.yaml`

File này định nghĩa quyền, nhưng chưa gắn quyền cho user. Có 3 object:

```text
Role developer
ClusterRole sre-pod-operator
ClusterRole cluster-viewer-readonly
```

### 1. `Role developer`

```yaml
kind: Role
metadata:
  name: developer
  namespace: demo
```

Đây là Role namespaced, chỉ có hiệu lực trong namespace `demo`.

Quyền được cấp:

- Core API group `""`: `pods`, `services`, `configmaps`.
- Apps API group `"apps"`: `deployments`, `replicasets`.
- Argo Rollouts API group `"argoproj.io"`: `rollouts`.
- Verbs: `get`, `list`, `watch`, `create`, `update`, `patch`, `delete`.

Ý nghĩa:

```text
alice có thể quản lý workload/app resource trong namespace demo.
alice không tự động có quyền ở kube-system hoặc namespace khác.
```

Vì sao dùng `Role` thay vì `ClusterRole` cho alice:

- Alice là developer của một namespace cụ thể.
- Scope nhỏ nhất cần thiết là namespace `demo`.
- Đây là least privilege: cấp vừa đủ, không cấp toàn cluster.

### 2. `ClusterRole sre-pod-operator`

```yaml
kind: ClusterRole
metadata:
  name: sre-pod-operator
```

Đây là ClusterRole, tức quyền có thể áp dụng toàn cluster nếu bind bằng `ClusterRoleBinding`.

Quyền được cấp:

- Với `pods`: `get`, `list`, `watch`, `delete`.
- Với `pods/log`: `get`, `list`.
- Với `pods/exec`: `create`.

Ý nghĩa:

```text
bob là SRE, cần xem pod toàn cluster, xem log, exec vào pod, delete pod để xử lý sự cố.
bob không có quyền với nodes, roles, secrets, deployments trừ khi được cấp thêm.
```

Chi tiết quan trọng:

- `pods/log` là subresource để đọc log.
- `pods/exec` dùng verb `create`, vì exec được Kubernetes model như tạo một phiên exec.
- Không cấp `nodes`, nên lệnh delete node phải bị từ chối.

### 3. `ClusterRole cluster-viewer-readonly`

```yaml
kind: ClusterRole
metadata:
  name: cluster-viewer-readonly
```

Quyền:

```yaml
apiGroups: ["*"]
resources: ["*"]
verbs: ["get", "list", "watch"]
```

Ý nghĩa:

```text
carol có thể đọc mọi resource trong cluster nhưng không được create/update/delete.
```

Vì sao chỉ có `get/list/watch`:

- `get`: đọc một object cụ thể.
- `list`: liệt kê object.
- `watch`: theo dõi thay đổi.
- Không có `create`, `update`, `patch`, `delete`, nên carol không thay đổi được cluster.

## File: `rbac/rolebindings.yaml`

File này gắn quyền đã định nghĩa ở `roles.yaml` cho user cụ thể.

### 1. `RoleBinding alice-developer-demo`

```yaml
kind: RoleBinding
metadata:
  name: alice-developer-demo
  namespace: demo
subjects:
- kind: User
  name: alice
roleRef:
  kind: Role
  name: developer
```

Ý nghĩa:

```text
Gắn Role developer trong namespace demo cho user alice.
```

Vì đây là `RoleBinding` namespaced:

- Alice có quyền trong `demo`.
- Alice không có quyền tương tự trong `kube-system`.
- Đây là lý do test `create deploy -n demo --as alice` trả `yes`, nhưng `-n kube-system` trả `no`.

### 2. `ClusterRoleBinding bob-sre-pod-operator`

```yaml
kind: ClusterRoleBinding
subjects:
- kind: User
  name: bob
roleRef:
  kind: ClusterRole
  name: sre-pod-operator
```

Ý nghĩa:

```text
Gắn ClusterRole sre-pod-operator cho bob trên toàn cluster.
```

Vì dùng `ClusterRoleBinding`, bob có thể thao tác pod ở mọi namespace. Nhưng bob chỉ có quyền đúng những resource trong ClusterRole, không phải admin.

### 3. `ClusterRoleBinding carol-cluster-viewer-readonly`

```yaml
kind: ClusterRoleBinding
subjects:
- kind: User
  name: carol
roleRef:
  kind: ClusterRole
  name: cluster-viewer-readonly
```

Ý nghĩa:

```text
Gắn quyền chỉ đọc toàn cluster cho carol.
```

Carol có thể đọc nodes, pods, services, deployments, v.v. Nhưng không thể delete nodes vì role không có verb `delete`.

## File: `argocd/apps/rbac.yaml`

Đây là ArgoCD Application sync thư mục `rbac/`.

```yaml
source:
  repoURL: https://github.com/kokoreina/template.git
  path: rbac
  targetRevision: main
destination:
  server: https://kubernetes.default.svc
  namespace: demo
```

Ý nghĩa:

```text
ArgoCD đọc manifest trong rbac/ từ branch main và apply vào cluster.
```

Các setting quan trọng:

- `automated.prune: true`: nếu resource bị xóa khỏi Git, ArgoCD xóa khỏi cluster.
- `automated.selfHeal: true`: nếu ai sửa tay trong cluster, ArgoCD kéo về đúng Git.
- `ServerSideApply=true`: dùng server-side apply, hữu ích với CRD/resource có schema phức tạp.
- Sync wave `"1"`: RBAC sync sau namespace common.

## Workflow Lab 1.1

```text
1. app-common tạo namespace demo
2. rbac app sync rbac/roles.yaml và rbac/rolebindings.yaml
3. Kubernetes tạo Role, ClusterRole, RoleBinding, ClusterRoleBinding
4. Bạn kiểm tra bằng kubectl auth can-i
```

Lệnh test:

```bash
kubectl auth can-i create deploy -n demo --as alice
kubectl auth can-i create deploy -n kube-system --as alice
kubectl auth can-i get pods -A --as bob
kubectl auth can-i delete nodes --as carol
```

Kỳ vọng:

```text
alice create deploy trong demo        = yes
alice create deploy trong kube-system = no
bob get pods all namespaces           = yes
carol delete nodes                    = no
```

Nếu kết quả sai, kiểm tra:

```bash
kubectl get role,rolebinding -n demo
kubectl get clusterrole | grep -E "sre|viewer"
kubectl get clusterrolebinding | grep -E "bob|carol"
kubectl describe role developer -n demo
kubectl describe rolebinding alice-developer-demo -n demo
```

---

# Lab 1.2 - Gatekeeper Admission Policies

Mục tiêu: cài OPA Gatekeeper và enforce 4 luật chặn manifest xấu tại admission time.

4 luật:

```text
1. Cấm image tag :latest
2. Bắt buộc resources.limits.cpu và resources.limits.memory
3. Cấm runAsUser: 0
4. Cấm hostNetwork: true
```

Điểm cần hiểu: Gatekeeper không phải scanner sau khi deploy. Gatekeeper là admission controller. Khi bạn apply Pod/Deployment, API server gọi webhook của Gatekeeper. Nếu policy vi phạm, request bị reject trước khi resource được tạo.

## File: `argocd/apps/gatekeeper.yaml`

ArgoCD Application này cài Gatekeeper controller bằng Helm chart.

```yaml
source:
  repoURL: https://open-policy-agent.github.io/gatekeeper/charts
  chart: gatekeeper
  targetRevision: 3.18.2
destination:
  namespace: gatekeeper-system
```

Ý nghĩa:

```text
ArgoCD dùng Helm chart upstream để cài Gatekeeper controller vào namespace gatekeeper-system.
```

Vì sao sync wave `-2`:

- Gatekeeper controller phải có trước.
- CRD như `ConstraintTemplate` phải được cài trước khi apply template/policy.
- Nếu apply policy trước controller/CRD, Kubernetes không nhận ra kind đó.

## File: `argocd/apps/gatekeeper-templates.yaml`

ArgoCD Application sync thư mục:

```yaml
path: gatekeeper/templates
```

Trong thư mục này có các `ConstraintTemplate`. ConstraintTemplate là "định nghĩa loại policy". Nó tạo ra một custom kind mới để sau đó bạn viết Constraint.

Ví dụ:

```text
k8sdisallowedlatesttag.yaml tạo kind K8sDisallowedLatestTag
k8srequiredlimits.yaml tạo kind K8sRequiredLimits
k8sdisallowrootuser.yaml tạo kind K8sDisallowRootUser
k8sdisallowhostnetwork.yaml tạo kind K8sDisallowHostNetwork
k8smaxreplicas.yaml tạo kind K8sMaxReplicas
```

Vì sao sync wave `-1`:

```text
Gatekeeper controller wave -2
ConstraintTemplate wave -1
Constraint wave 1
```

## File: `argocd/apps/gatekeeper-constraints.yaml`

ArgoCD Application sync thư mục:

```yaml
path: gatekeeper/constraints
```

Trong thư mục này có các Constraint thật sự bật deny.

Setting quan trọng:

```yaml
syncOptions:
- SkipDryRunOnMissingResource=true
```

Vì sao cần:

- ArgoCD dry-run có thể chạy trước khi CRD từ ConstraintTemplate sẵn sàng.
- Option này giúp tránh lỗi dry-run khi kind custom vừa mới được tạo.

## ConstraintTemplate Là Gì?

ConstraintTemplate gồm hai phần:

```text
1. CRD schema: tạo ra kind policy mới
2. Rego code: logic kiểm tra vi phạm
```

Gatekeeper sẽ nhận object Kubernetes trong biến `input.review.object`. Rego đọc object đó và trả về `violation` nếu có lỗi.

Các template trong repo dùng hàm `get_pod_spec` để xử lý nhiều loại workload:

```text
Pod        -> obj.spec
CronJob    -> obj.spec.jobTemplate.spec.template.spec
Deployment -> obj.spec.template.spec
StatefulSet -> obj.spec.template.spec
DaemonSet -> obj.spec.template.spec
Rollout    -> obj.spec.template.spec
```

Nhờ vậy cùng một policy có thể kiểm tra cả Pod trực tiếp và workload controller.

## File: `gatekeeper/templates/k8sdisallowedlatesttag.yaml`

Tạo kind:

```yaml
kind: K8sDisallowedLatestTag
```

Logic Rego chính:

```rego
violation[{"msg": msg}] {
  c := all_containers[_]
  endswith(c.image, ":latest")
  msg := sprintf("container %v uses forbidden image tag :latest (%v)", [c.name, c.image])
}
```

Ý nghĩa:

```text
Nếu container image kết thúc bằng :latest, reject.
```

Vì sao cấm `:latest`:

- `latest` không immutable.
- Hôm nay và ngày mai cùng tag nhưng có thể trỏ image khác.
- Rollback/debug khó vì không biết chính xác version nào đang chạy.
- GitOps cần manifest khai báo trạng thái mong muốn cụ thể.

## File: `gatekeeper/templates/k8srequiredlimits.yaml`

Tạo kind:

```yaml
kind: K8sRequiredLimits
```

Logic chính:

```rego
not c.resources.limits.cpu
not c.resources.limits.memory
```

Ý nghĩa:

```text
Mọi container/initContainer phải khai báo resources.limits.cpu và resources.limits.memory.
```

Vì sao cần limits:

- Tránh workload ăn hết CPU/RAM node.
- Scheduler và kubelet có cơ sở kiểm soát tài nguyên.
- Platform có guardrail để team không deploy pod "vô hạn".

Lưu ý: policy hiện tại chỉ bắt buộc `limits`, không bắt buộc `requests`.

## File: `gatekeeper/templates/k8sdisallowrootuser.yaml`

Tạo kind:

```yaml
kind: K8sDisallowRootUser
```

Logic chính:

```rego
spec.securityContext.runAsUser == 0
c.securityContext.runAsUser == 0
```

Ý nghĩa:

```text
Nếu Pod-level hoặc container-level securityContext set runAsUser: 0 thì reject.
```

Vì sao cấm root:

- Root trong container làm tăng blast radius nếu app bị khai thác.
- Dễ ghi file hệ thống trong image/container.
- Kết hợp với misconfig khác có thể dẫn tới privilege escalation.

Lưu ý kỹ:

- Policy này chặn khi manifest khai báo rõ `runAsUser: 0`.
- Nó không tự đảm bảo image không chạy root nếu manifest không khai báo `runAsNonRoot`.
- Muốn chặt hơn có thể viết thêm policy yêu cầu `runAsNonRoot: true`.

## File: `gatekeeper/templates/k8sdisallowhostnetwork.yaml`

Tạo kind:

```yaml
kind: K8sDisallowHostNetwork
```

Logic chính:

```rego
spec.hostNetwork == true
```

Ý nghĩa:

```text
Nếu workload dùng hostNetwork: true thì reject.
```

Vì sao cấm hostNetwork:

- Pod dùng network namespace của node.
- Có thể bypass một số isolation/network policy.
- Dễ đụng port host.
- Không nên cho app thường dùng trừ trường hợp system daemon đặc biệt.

## File: `gatekeeper/constraints/constraints.yaml`

Đây là file bật các policy ở chế độ deny.

### Constraint `deny-image-latest-tag`

```yaml
kind: K8sDisallowedLatestTag
metadata:
  name: deny-image-latest-tag
spec:
  enforcementAction: deny
```

`match.kinds` áp dụng cho:

- Pod
- Deployment, StatefulSet, DaemonSet
- Job, CronJob
- Rollout

`excludedNamespaces` bỏ qua các namespace hạ tầng:

```text
kube-system, kube-public, kube-node-lease, argocd,
gatekeeper-system, monitoring, argo-rollouts
```

Vì sao exclude:

- Namespace hệ thống thường có manifest từ chart/vendor.
- Nếu enforce nhầm có thể làm hỏng control plane/platform.
- App namespace như `demo` vẫn bị enforce.

### Constraint `require-cpu-memory-limits`

Bật kind `K8sRequiredLimits` ở `deny`.

Ý nghĩa:

```text
Mọi Pod/workload trong namespace không bị exclude phải có limits CPU và memory.
```

### Constraint `deny-runasuser-root`

Bật kind `K8sDisallowRootUser` ở `deny`.

Ý nghĩa:

```text
Không cho manifest khai runAsUser: 0.
```

### Constraint `deny-hostnetwork`

Bật kind `K8sDisallowHostNetwork` ở `deny`.

Ý nghĩa:

```text
Không cho Pod/workload dùng hostNetwork: true.
```

## Test Manifests Lab 1.2

Các file trong `gatekeeper/tests/` dùng để chứng minh policy hoạt động.

### `gatekeeper/tests/bad-latest.yaml`

```yaml
image: nginx:latest
```

Kỳ vọng:

```text
Reject vì vi phạm deny-image-latest-tag.
```

Command:

```bash
kubectl apply -f gatekeeper/tests/bad-latest.yaml
```

### `gatekeeper/tests/bad-no-limits.yaml`

Pod dùng `nginx:1.27` nhưng không có:

```yaml
resources:
  limits:
    cpu: ...
    memory: ...
```

Kỳ vọng:

```text
Reject vì vi phạm require-cpu-memory-limits.
```

### `gatekeeper/tests/bad-root.yaml`

Pod set:

```yaml
securityContext:
  runAsUser: 0
```

Kỳ vọng:

```text
Reject vì vi phạm deny-runasuser-root.
```

### `gatekeeper/tests/bad-hostnetwork.yaml`

Pod set:

```yaml
hostNetwork: true
```

Kỳ vọng:

```text
Reject vì vi phạm deny-hostnetwork.
```

### `gatekeeper/tests/good-pod.yaml`

Pod hợp lệ:

```yaml
image: nginx:1.27
securityContext:
  runAsUser: 1000
resources:
  limits:
    cpu: 100m
    memory: 64Mi
```

Kỳ vọng:

```text
Pass vì không dùng latest, có limits, không chạy root, không hostNetwork.
```

## Workflow Lab 1.2

```text
1. ArgoCD sync gatekeeper.yaml
2. Helm chart cài Gatekeeper controller + CRDs
3. ArgoCD sync gatekeeper-templates.yaml
4. Gatekeeper tạo custom kinds từ ConstraintTemplate
5. ArgoCD sync gatekeeper-constraints.yaml
6. Các constraints bật enforcementAction: deny
7. Khi apply workload, API server gọi Gatekeeper webhook
8. Gatekeeper chạy Rego
9. Nếu có violation -> API server reject request
10. Nếu không có violation -> resource được tạo
```

Lệnh kiểm tra controller:

```bash
kubectl get pods -n gatekeeper-system
kubectl get constrainttemplates
kubectl get constraints
```

Lệnh test deny/pass:

```bash
kubectl apply -f gatekeeper/tests/bad-latest.yaml
kubectl apply -f gatekeeper/tests/bad-no-limits.yaml
kubectl apply -f gatekeeper/tests/bad-root.yaml
kubectl apply -f gatekeeper/tests/bad-hostnetwork.yaml
kubectl apply -f gatekeeper/tests/good-pod.yaml
kubectl delete -f gatekeeper/tests/good-pod.yaml
```

Kỳ vọng:

```text
bad-latest       = reject
bad-no-limits    = reject
bad-root         = reject
bad-hostnetwork  = reject
good-pod         = created
```

---

# Lab 1.3 - Custom Policy: Max Replicas

Mục tiêu: tự viết custom ConstraintTemplate bằng Rego, không chỉ copy policy có sẵn. Repo chọn option:

```text
Reject Deployment/StatefulSet/Rollout nếu replicas > 5
```

## File: `gatekeeper/templates/k8smaxreplicas.yaml`

Tạo kind:

```yaml
kind: K8sMaxReplicas
```

Phần schema:

```yaml
validation:
  openAPIV3Schema:
    type: object
    properties:
      maxReplicas:
        type: integer
```

Ý nghĩa:

```text
Constraint dùng template này được phép truyền parameter maxReplicas kiểu integer.
```

Logic Rego:

```rego
violation[{"msg": msg}] {
  max := input.parameters.maxReplicas
  replicas := input.review.object.spec.replicas
  replicas > max
  msg := sprintf("%v/%v has replicas %v, max allowed is %v", [input.review.object.kind, input.review.object.metadata.name, replicas, max])
}
```

Giải thích từng dòng:

- `input.parameters.maxReplicas`: lấy giá trị từ Constraint.
- `input.review.object.spec.replicas`: đọc số replica trong Deployment/StatefulSet/Rollout đang được apply.
- `replicas > max`: điều kiện vi phạm.
- `msg := sprintf(...)`: tạo message trả về cho user.

Khi violation tồn tại, Gatekeeper reject request.

## File: `gatekeeper/constraints/constraints.yaml` phần `max-replicas-five`

Constraint:

```yaml
kind: K8sMaxReplicas
metadata:
  name: max-replicas-five
spec:
  enforcementAction: deny
  parameters:
    maxReplicas: 5
```

Match:

```yaml
kinds:
- apiGroups: ["apps"]
  kinds: ["Deployment", "StatefulSet"]
- apiGroups: ["argoproj.io"]
  kinds: ["Rollout"]
```

Ý nghĩa:

```text
Chỉ enforce replica limit cho Deployment, StatefulSet, Rollout.
Không enforce Pod trực tiếp vì Pod không có spec.replicas.
```

Vì sao `parameters.maxReplicas: 5`:

- Template tái sử dụng được.
- Nếu sau này muốn max 10, chỉ sửa Constraint parameter, không sửa Rego.

## File: `gatekeeper/tests/bad-replicas.yaml`

Deployment test:

```yaml
kind: Deployment
metadata:
  name: bad-replicas
  namespace: demo
spec:
  replicas: 6
```

Kỳ vọng:

```text
Reject vì replicas 6 > maxReplicas 5.
```

Command:

```bash
kubectl apply -f gatekeeper/tests/bad-replicas.yaml
```

## Workflow Lab 1.3

```text
1. ArgoCD sync gatekeeper/templates/k8smaxreplicas.yaml
2. Gatekeeper tạo custom kind K8sMaxReplicas
3. ArgoCD sync gatekeeper/constraints/constraints.yaml
4. Constraint max-replicas-five bật deny với maxReplicas = 5
5. Khi user apply Deployment/StatefulSet/Rollout, Gatekeeper đọc spec.replicas
6. Nếu replicas > 5 -> reject
7. Nếu replicas <= 5 -> pass qua policy này
```

Test:

```bash
kubectl apply -f gatekeeper/tests/bad-replicas.yaml
```

Kỳ vọng output có ý nghĩa tương tự:

```text
admission webhook "validation.gatekeeper.sh" denied the request:
Deployment/bad-replicas has replicas 6, max allowed is 5
```

---

# File Chung Phục Vụ Lab 1

## File: `argocd/root.yaml`

Đây là root Application.

```yaml
source:
  repoURL: https://github.com/kokoreina/template.git
  path: argocd/apps
  targetRevision: main
```

Ý nghĩa:

```text
Root app không deploy workload trực tiếp.
Root app deploy các ArgoCD Application con trong argocd/apps/.
```

Workflow:

```text
kubectl apply -f argocd/root.yaml
-> ArgoCD tạo root app
-> root app đọc argocd/apps/
-> tạo/sync các child app
-> child app sync rbac/, gatekeeper/, app-common/, v.v.
```

## File: `app-common/demo-namespace.yaml`

Tạo namespace `demo`.

```yaml
kind: Namespace
metadata:
  name: demo
```

Vì sao cần:

- RBAC Role `developer` nằm trong namespace `demo`.
- Các test pod Gatekeeper cũng deploy vào namespace `demo`.
- App API/Rollout của lab khác cũng dùng namespace này.

## File: `argocd/apps/app-common.yaml`

ArgoCD Application sync thư mục `app-common/`.

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
```

Ý nghĩa:

```text
Namespace demo được tạo sớm để các app sau không lỗi namespace missing.
```

---

# Luồng Nghiệm Thu Đầy Đủ Lab 1.1 - 1.3

## Bước 1: Apply root app

```bash
kubectl apply -f argocd/root.yaml
```

Kiểm tra ArgoCD:

```bash
kubectl get applications -n argocd
```

Kỳ vọng các app liên quan:

```text
common
rbac
gatekeeper
gatekeeper-templates
gatekeeper-constraints
```

## Bước 2: Kiểm tra namespace và RBAC

```bash
kubectl get ns demo
kubectl get role,rolebinding -n demo
kubectl get clusterrole | grep -E "sre|viewer"
kubectl get clusterrolebinding | grep -E "bob|carol"
```

RBAC evidence:

```bash
kubectl auth can-i create deploy -n demo --as alice
kubectl auth can-i create deploy -n kube-system --as alice
kubectl auth can-i get pods -A --as bob
kubectl auth can-i delete nodes --as carol
```

Expected:

```text
yes
no
yes
no
```

## Bước 3: Kiểm tra Gatekeeper đã chạy

```bash
kubectl get pods -n gatekeeper-system
kubectl get constrainttemplates
kubectl get constraints
```

Bạn nên thấy:

```text
k8sdisallowedlatesttag
k8srequiredlimits
k8sdisallowrootuser
k8sdisallowhostnetwork
k8smaxreplicas
```

Và constraints:

```text
deny-image-latest-tag
require-cpu-memory-limits
deny-runasuser-root
deny-hostnetwork
max-replicas-five
```

## Bước 4: Test các manifest vi phạm

```bash
kubectl apply -f gatekeeper/tests/bad-latest.yaml
kubectl apply -f gatekeeper/tests/bad-no-limits.yaml
kubectl apply -f gatekeeper/tests/bad-root.yaml
kubectl apply -f gatekeeper/tests/bad-hostnetwork.yaml
kubectl apply -f gatekeeper/tests/bad-replicas.yaml
```

Kỳ vọng:

```text
Tất cả bị reject bởi Gatekeeper.
```

## Bước 5: Test manifest hợp lệ

```bash
kubectl apply -f gatekeeper/tests/good-pod.yaml
kubectl get pod good-pod -n demo
kubectl delete -f gatekeeper/tests/good-pod.yaml
```

Kỳ vọng:

```text
good-pod được tạo thành công.
```

---

# Tư Duy Thiết Kế Của Lab 1

## RBAC và Gatekeeper khác nhau thế nào?

RBAC trả lời:

```text
Ai được gọi API nào?
```

Ví dụ:

```text
alice có được create deployment trong demo không?
```

Gatekeeper trả lời:

```text
Object được gửi lên API server có hợp lệ theo policy không?
```

Ví dụ:

```text
Deployment này có dùng :latest không?
Pod này có thiếu limits không?
Pod này có chạy root không?
```

Nói ngắn:

```text
RBAC = kiểm soát quyền của người gọi
Gatekeeper = kiểm soát chất lượng/an toàn của manifest
```

## Vì sao dùng GitOps?

Lab yêu cầu "mọi thứ qua Git", không `kubectl apply` tay cho resource chính.

Lợi ích:

- Git là source of truth.
- Có lịch sử thay đổi.
- Có thể review bằng pull request.
- ArgoCD tự sync/self-heal.
- Dễ chứng minh với mentor: manifest nằm trong repo, cluster khớp repo.

## Vì sao phải dùng sync wave?

Vì Kubernetes resource có thứ tự phụ thuộc:

```text
Namespace trước RoleBinding/workload
Gatekeeper controller trước ConstraintTemplate
ConstraintTemplate trước Constraint
Constraint trước test reject
```

Nếu sai thứ tự, thường gặp lỗi:

```text
namespace not found
no matches for kind "K8sRequiredLimits"
no matches for kind "ConstraintTemplate"
```

## Vì sao test có cả bad và good?

Bad test chứng minh policy thật sự chặn được manifest xấu.

Good test chứng minh policy không chặn nhầm manifest hợp lệ.

Một policy tốt phải làm được cả hai:

```text
Vi phạm -> reject
Hợp lệ  -> pass
```

---

# Cheat Sheet Lệnh Nhanh

Apply App of Apps:

```bash
kubectl apply -f argocd/root.yaml
```

Xem app:

```bash
kubectl get applications -n argocd
```

RBAC:

```bash
kubectl auth can-i create deploy -n demo --as alice
kubectl auth can-i create deploy -n kube-system --as alice
kubectl auth can-i get pods -A --as bob
kubectl auth can-i delete nodes --as carol
```

Gatekeeper:

```bash
kubectl get pods -n gatekeeper-system
kubectl get constrainttemplates
kubectl get constraints
```

Policy tests:

```bash
kubectl apply -f gatekeeper/tests/bad-latest.yaml
kubectl apply -f gatekeeper/tests/bad-no-limits.yaml
kubectl apply -f gatekeeper/tests/bad-root.yaml
kubectl apply -f gatekeeper/tests/bad-hostnetwork.yaml
kubectl apply -f gatekeeper/tests/bad-replicas.yaml
kubectl apply -f gatekeeper/tests/good-pod.yaml
kubectl delete -f gatekeeper/tests/good-pod.yaml
```

Nếu cần xem chi tiết lỗi reject:

```bash
kubectl apply -f gatekeeper/tests/bad-latest.yaml -v=6
```

Nếu cần debug một constraint:

```bash
kubectl describe constrainttemplate k8srequiredlimits
kubectl describe k8srequiredlimits require-cpu-memory-limits
```
