---
doc_id: vault-index
title: Vault Index
type: index
domain: vault
keywords:
  - index
  - catalog
  - naming
  - query
status: active
---

# Vault Index

## Mục tiêu

- Vault này ưu tiên `folder + slug + frontmatter` để dễ query và dễ rename về sau.
- `Index.md` không dùng `[[wikilink]]` và không tạo internal link giữa các note.
- Khi cần tìm tài liệu, ưu tiên search theo `doc_id`, `title`, `legacy_title`, `type`, `domain` hoặc `keywords`.

## Cấu trúc thư mục

```text
01-concepts/
  kubernetes/
02-architecture/
  postgresql-ha/
03-runbooks/
  longhorn/
99-scratch/
```

## Quy ước đặt tên

- Folder biểu diễn loại tài liệu, không biểu diễn ngày hoặc trạng thái tạm thời.
- File name dùng slug ASCII, lowercase và dấu `-` để dễ grep, script và truy vấn path.
- `title` giữ tên đọc được cho con người; `doc_id` giữ định danh ổn định để không phụ thuộc file name.
- `legacy_title` giữ tên cũ để sau này đổi tên file vẫn search ngược được.
- Không dùng `[[...]]` trong index hoặc phần liên hệ; nếu cần tham chiếu chéo, ghi `doc_id` hoặc path dưới dạng code.

## Catalog tài liệu

| Folder | File | Title | Type | Query keys |
|---|---|---|---|---|
| `01-concepts/kubernetes` | `configmap-secret-mount-into-pod.md` | ConfigMap, Secret và cơ chế mount vào Pod | `concept-note` | `configmap`, `secret`, `mount`, `env`, `volume`, `pod` |
| `01-concepts/kubernetes` | `db-deployment-pvc-vs-statefulset.md` | DB Deployment + PVC vs DB StatefulSet | `concept-note` | `database`, `deployment`, `pvc`, `statefulset`, `storage` |
| `01-concepts/kubernetes` | `kubernetes-architecture-explained.md` | Kiến trúc Kubernetes từ control plane đến worker node | `concept-note` | `kubernetes`, `architecture`, `control-plane`, `worker-node`, `kubelet`, `etcd`, `scheduler` |
| `01-concepts/kubernetes` | `pod-priority-preemption-pdb-qos.md` | Pod Priority, Preemption, PDB và QoS trong Kubernetes | `concept-note` | `pod-priority`, `priorityclass`, `preemption`, `pdb`, `qos`, `eviction`, `scheduler` |
| `01-concepts/kubernetes` | `raft-etcd-quorum-vs-repmgr.md` | RAFT, etcd quorum và khác biệt với repmgr | `concept-note` | `raft`, `etcd`, `quorum`, `leader-election`, `consensus`, `repmgr` |
| `02-architecture/postgresql-ha` | `bitnami-images-in-postgresql-statefulset.md` | Bitnami Images trong PostgreSQL StatefulSet | `reference-note` | `postgresql`, `bitnami`, `image`, `statefulset`, `high-availability` |
| `02-architecture/postgresql-ha` | `postgresql-ha-statefulset-longhorn-repmgr.md` | PostgreSQL HA trên Kubernetes - StatefulSet, Longhorn, repmgr | `architecture-note` | `postgresql`, `high-availability`, `statefulset`, `longhorn`, `repmgr` |
| `03-runbooks/longhorn` | `longhorn-network-iscsi-node3-debug-runbook.md` | Runbook debug Longhorn -> Network -> iSCSI khi node3 không bắt được Longhorn | `runbook` | `longhorn`, `network`, `iscsi`, `node3`, `csi`, `debug` |
| `99-scratch` | `k8s-manual-commands.md` | K8s Manual Commands | `scratch-note` | `commands`, `manual`, `scratch` |

## Thứ tự đọc gợi ý

1. `01-concepts/kubernetes/kubernetes-architecture-explained.md`
Vai trò: note nền để nắm cấu trúc control plane, worker node, networking và workflow tổng thể của cluster.

2. `01-concepts/kubernetes/raft-etcd-quorum-vs-repmgr.md`
Vai trò: note nền để hiểu vì sao etcd dùng quorum, RAFT commit như thế nào, và nó khác repmgr ở đâu.

3. `02-architecture/postgresql-ha/postgresql-ha-statefulset-longhorn-repmgr.md`
Vai trò: note trục chính để hiểu 3 lớp Kubernetes, storage và database.

4. `01-concepts/kubernetes/db-deployment-pvc-vs-statefulset.md`
Vai trò: phân biệt workload stateful và cách chọn controller cho database.

5. `01-concepts/kubernetes/configmap-secret-mount-into-pod.md`
Vai trò: hiểu cách truyền cấu hình vào Pod trước khi đọc manifest HA chi tiết.

6. `01-concepts/kubernetes/pod-priority-preemption-pdb-qos.md`
Vai trò: hiểu rõ quan hệ giữa scheduler priority, preemption, PDB và QoS khi cluster thiếu tài nguyên.

7. `02-architecture/postgresql-ha/bitnami-images-in-postgresql-statefulset.md`
Vai trò: giải thích vì sao tutorial PostgreSQL HA dùng image Bitnami và script sẵn có.

8. `03-runbooks/longhorn/longhorn-network-iscsi-node3-debug-runbook.md`
Vai trò: tài liệu debug thực chiến cho case Longhorn / network / iSCSI.

## Quy tắc rename an toàn

- Nếu chỉ muốn đổi tên hiển thị, sửa `title` trong frontmatter trước.
- Nếu muốn đổi tên file, giữ nguyên `doc_id` để query và script không bị gãy.
- Nếu note từng có tên cũ quan trọng, cập nhật `legacy_title` hoặc thêm từ khóa vào `keywords`.
- Với note chưa hoàn chỉnh, chuyển vào `99-scratch` thay vì để chung với note chuẩn.

## Cách truy vấn nhanh

- Tìm theo chủ đề: search `keywords: longhorn`, `keywords: statefulset`, `domain: kubernetes`.
- Tìm theo loại tài liệu: search `type: runbook`, `type: concept-note`, `type: architecture-note`.
- Tìm theo tên cũ: search `legacy_title: Runbook debug Longhorn - Network - iSCSI node3`.
- Tìm theo path: search `03-runbooks/longhorn` hoặc `02-architecture/postgresql-ha`.
