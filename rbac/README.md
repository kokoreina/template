# Lab 1.1 - RBAC

## Vai trò

- `alice`: developer, CRUD workload trong namespace `demo`.
- `bob`: SRE, xem và thao tác pod toàn cluster.
- `carol`: viewer, chỉ đọc toàn cluster.

## Nghiệm thu

```bash
kubectl auth can-i create deploy -n demo --as alice        # yes
kubectl auth can-i create deploy -n kube-system --as alice # no
kubectl auth can-i get pods -A --as bob                    # yes
kubectl auth can-i delete nodes --as carol                 # no
```
