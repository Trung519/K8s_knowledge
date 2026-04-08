---
doc_id: postgresql-ha-statefulset-longhorn-repmgr
title: PostgreSQL HA trên Kubernetes - StatefulSet, Longhorn, repmgr
legacy_title: PostgreSQL HA trên Kubernetes - StatefulSet Longhorn repmgr
type: architecture-note
domain: kubernetes
keywords:
  - postgresql
  - high-availability
  - statefulset
  - longhorn
  - repmgr
status: active
---

# PostgreSQL HA trên Kubernetes - StatefulSet, Longhorn, repmgr

## Nguồn tham chiếu

- Chat gốc: [ChatGPT Shared Conversation](https://chatgpt.com/share/69d3eb27-4be8-83a1-92d6-26480dfede05)
- Kubernetes StatefulSet: [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- Kubernetes single-instance stateful app: [Run a Single-Instance Stateful Application](https://kubernetes.io/docs/tasks/run-application/run-single-instance-stateful-application/)
- Kubernetes replicated stateful app: [Run a Replicated Stateful Application](https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/)
- Kubernetes scheduling: [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- Longhorn docs: [Longhorn Documentation](https://longhorn.io/docs/latest/)
- repmgr: [repmgr - Replication Manager for PostgreSQL clusters](https://www.repmgr.org/)
- PostgreSQL HA: [High Availability, Load Balancing, and Replication](https://www.postgresql.org/docs/current/high-availability.html)
- Bitnami PostgreSQL chart: [bitnami/postgresql README](https://github.com/bitnami/charts/blob/main/bitnami/postgresql/README.md)
- Bitnami PostgreSQL HA chart: [bitnami/postgresql-ha README](https://github.com/bitnami/charts/blob/main/bitnami/postgresql-ha/README.md)
- MySQL Operator reference: [MySQL Operator for Kubernetes PDF](https://downloads.mysql.com/docs/mysql-operator-en.pdf)

## Kết luận 1 dòng

==StatefulSet không đồng bộ dữ liệu, Longhorn không quản lý role database, repmgr không thay StatefulSet; ba thứ này nằm ở 3 lớp khác nhau và bổ trợ cho nhau.==

## Bản đồ 3 lớp

### 1. Kubernetes layer

- `StatefulSet` giữ identity ổn định cho từng Pod như `db-0`, `db-1`, `db-2`
- giữ network identity và storage identity ổn định
- quản lý scale, rollout, recreate theo thứ tự phù hợp workload stateful

### 2. Storage layer

- `Longhorn` cấp volume persistent cho workload
- replicate volume sang nhiều node ở tầng storage
- snapshot, backup, restore ở tầng volume

### 3. Database layer

- `PostgreSQL` tự lo replication ở tầng database
- `repmgr` giúp setup, monitor, switchover, failover cho PostgreSQL cluster

## Những điều rất dễ nhầm

### StatefulSet không phải công cụ sync data

`replicas: 3` trong StatefulSet chỉ có nghĩa là Kubernetes có 3 Pod, không có nghĩa là 3 database node tự động replicate cho nhau.

### Replica của Kubernetes khác replica của database

- Replica của Kubernetes = số lượng Pod
- Replica của database = bản sao dữ liệu, role primary/standby, cơ chế WAL/streaming replication

### Longhorn không thay thế replication database

Longhorn chỉ biết volume block storage. Nó không hiểu `primary`, `standby`, transaction hay WAL của PostgreSQL.

### repmgr không thay thế StatefulSet

repmgr xử lý role cluster và failover. Nhưng nếu không có nền tảng identity/storage ổn định cho từng node database, việc vận hành cluster sẽ rất khó.

## StatefulSet thực sự giải quyết bài toán nào

- mỗi Pod có tên riêng và ổn định
- mỗi Pod có PVC riêng thông qua `volumeClaimTemplates`
- Pod bị reschedule vẫn quay lại đúng danh tính và đúng volume của nó
- phù hợp cho database cluster, Kafka, Cassandra, Elasticsearch, MongoDB replica set

## Vì sao mỗi Pod phải có PVC riêng

Với database HA đúng nghĩa, mỗi node có data directory riêng. Đồng bộ dữ liệu diễn ra qua replication của database, không phải bằng cách nhiều Pod cùng ghi vào một volume chung.

Ví dụ:

- `postgres-0` -> `data-postgres-0`
- `postgres-1` -> `data-postgres-1`
- `postgres-2` -> `data-postgres-2`

## StatefulSet có tự động đặt 3 Pod lên 3 node khác nhau không

Không. Nếu muốn tăng HA theo node/zone, cần thêm:

- `nodeAffinity`
- `podAntiAffinity`
- `topologySpreadConstraints`

## Khi nào dùng Deployment, khi nào dùng StatefulSet

### Nên dùng Deployment

- app stateless
- web/API chỉ gọi DB bình thường
- single-instance database đơn giản trong dev/test/lab

### Nên dùng StatefulSet

- database cluster
- workload cần hostname ổn định
- cần PVC riêng cho từng replica
- cần ordered rollout/termination

## Có Longhorn mà không có repmgr thì sao

Bạn mới có **storage HA**, chưa có **database HA đúng nghĩa**. Volume có thể bền hơn khi node gặp sự cố, nhưng PostgreSQL vẫn không tự có failover primary/standby.

## Có repmgr mà không có Longhorn thì sao

Vẫn có thể có **database HA**, vì replication/failover là việc của PostgreSQL/repmgr. Longhorn chỉ là một lựa chọn storage backend trong Kubernetes.

## Kiến trúc nên nhớ

```text
App (Deployment)
  -> Service/PostgreSQL endpoint
    -> PostgreSQL cluster
      -> StatefulSet giữ identity + PVC riêng cho từng member
      -> Storage có thể được cấp bởi Longhorn
      -> Replication/failover do PostgreSQL + repmgr xử lý
```

## 5 câu học thuộc

1. `StatefulSet` không đồng bộ dữ liệu; nó giữ identity và storage ổn định cho từng Pod.
2. `replicas` của Kubernetes không đồng nghĩa với replica dữ liệu của database.
3. `Longhorn` đồng bộ volume ở tầng storage, không đồng bộ logic database.
4. `repmgr` quản lý replication và failover cho PostgreSQL cluster.
5. Muốn PostgreSQL HA đúng nghĩa trên Kubernetes thì chỉ dùng StatefulSet là chưa đủ.

## Nhanh gọn để phân biệt

| Thành phần | Trả lời câu hỏi nào |
|---|---|
| `StatefulSet` | Mỗi node DB là ai, tên gì, gắn volume nào, khởi tạo/xóa theo thứ tự nào? |
| `Longhorn` | Volume lưu ở đâu, replicate volume ra sao, backup/snapshot thế nào? |
| `PostgreSQL + repmgr` | Node nào là primary, node nào là standby, failover thế nào? |

## Note thực chiến

- Nếu chỉ lấy `postgres` image thông thường, bỏ vào `StatefulSet`, rồi scale `replicas > 1` thì không tự động thành PostgreSQL HA cluster.
- Production thường dùng chart HA, operator, hoặc image đã đóng gói sẵn logic cluster thay vì chỉ viết một StatefulSet cơ bản.
- Cần nghĩ theo 3 lớp: workload, storage, database. Nhiều tranh cãi xảy ra vì trộn 3 lớp này vào nhau.

## Mở rộng để học tiếp

- Tìm hiểu thêm `PodDisruptionBudget`, `readinessProbe`, `livenessProbe`, `preStop`
- Tìm hiểu service routing cho `read/write split`
- Tìm hiểu operator như CloudNativePG, Crunchy, MySQL Operator
