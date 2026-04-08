---
doc_id: bitnami-images-in-postgresql-statefulset
title: Bitnami Images trong PostgreSQL StatefulSet
legacy_title: Bitnami Images trong PostgreSQL StatefulSet
type: reference-note
domain: kubernetes
keywords:
  - postgresql
  - bitnami
  - image
  - statefulset
  - high-availability
status: active
---

# Bitnami Images trong PostgreSQL StatefulSet

Nguồn đọc:
- Bài gốc: [How to Deploy PostgreSQL Statefulset in Kubernetes With High Availability](https://devopscube.com/deploy-postgresql-statefulset/)
- Tham khảo thêm: [Bitnami Containers](https://bitnami.com/stacks/containers)

## Bitnami images là gì?

==`Bitnami image` là Docker container image do Bitnami đóng gói sẵn cho các phần mềm phổ biến như PostgreSQL, Redis, Nginx, Kafka, WordPress.==

Ý chính là:
- ==Bạn không cần tự cài đặt tay từng gói trong image.==
- ==Nhiều cấu hình, script khởi tạo, biến môi trường và best practice đã được đóng gói sẵn.==
- ==Bạn chỉ cần kéo image về và cấu hình bằng `env`, `Secret`, `ConfigMap`, `volume` và manifest Kubernetes.==

Nói ngắn gọn: ==Bitnami images là các image "đóng gói sẵn, tài liệu rõ ràng, dễ dùng"== để giúp mình tập trung vào việc deploy và vận hành thay vì tự build image từ đầu.

## Vì sao bài này dùng Bitnami?

Trong bài viết, tác giả nói rõ tutorial cố ý dùng Bitnami image vì:
- ==image đã có sẵn các thành phần cần thiết==
- ==đã được test/validate trước khi release==
- ==tài liệu env vars khá đầy đủ, hợp cho người mới học==

Điều này rất hợp với bài học Kubernetes, vì ==mục tiêu chính là hiểu `StatefulSet`, `Service`, `Secret`, `ConfigMap`, replication và failover==, thay vì mất thời gian build image PostgreSQL từ đầu.

## Ví dụ ngay trong bài PostgreSQL StatefulSet

Trong bài này, Postgres không chỉ là "một container chạy postgres". Image Bitnami còn kèm theo:
- ==script helper trong `/opt/bitnami/scripts/...`==
- ==các biến môi trường để cấu hình PostgreSQL và `repmgr`==
- ==cơ chế phục vụ replication/failover để dùng trong mô hình HA==

Ví dụ, script `pre-stop.sh` trong bài có source các thư viện sau:

```bash
. /opt/bitnami/scripts/liblog.sh
. /opt/bitnami/scripts/libpostgresql.sh
. /opt/bitnami/scripts/librepmgr.sh
```

Điều này cho thấy ==image Bitnami đã đóng gói sẵn logic hỗ trợ PostgreSQL và `repmgr`==, nên pod có thể:
- nhận biết node nào đang là primary
- dừng PostgreSQL đúng cách trước khi pod bị stop
- chờ failover xong rồi mới rời cluster nếu node primary đang tắt

<span style="color:red">Điểm rất quan trọng: Bitnami không chỉ cung cấp một image "có PostgreSQL", mà còn cung cấp cả hệ script và cơ chế hỗ trợ để chạy cụm PostgreSQL HA dễ hơn.</span>

Nếu bạn tự dùng một image postgres rất "thuần" hoặc tự build từ `ubuntu`, bạn sẽ phải tự cài thêm package, tự viết script khởi tạo, tự xử lý replication, tự viết health/failover logic.

<span style="color:red">Nếu không có lớp đóng gói này, phần khó không nằm ở Kubernetes mà nằm ở việc tự chuẩn bị ứng dụng database cho đúng cách.</span>

## Một ví dụ để hiểu dễ hơn

Hãy so sánh 2 cách:

### 1. Dùng image thường, tự build

Bạn có thể phải tự làm:
- cài ==PostgreSQL==``
- cài `repmgr`
- viết entrypoint script
- viết logic init database
- viết script shutdown graceful
- viết tài liệu cho team dùng

### 2. Dùng `bitnami/postgresql`

Bạn thường chỉ cần:
- ==tham chiếu image Bitnami==
- ==truyền password qua `Secret`==
- ==mount script/cấu hình qua `ConfigMap`==
- ==cấp `PersistentVolume`==
- ==deploy bằng `StatefulSet`==

Nghĩa là ==Bitnami không thay thế Kubernetes==, mà giúp giảm phần việc "đóng gói ứng dụng" để mình tập trung vào phần "vận hành trên Kubernetes".

## Khi nào nên dùng Bitnami images?

Bitnami rất hợp khi:
- bạn đang học Kubernetes
- cần PoC, lab, dev, staging nhanh
- muốn có image đã được đóng gói và tài liệu hóa tương đối tốt
- muốn giảm công sức tự build image riêng

## Khi nào cần cân nhắc kỹ?

Vẫn có lúc bạn nên tự build image riêng, ví dụ:
- cần hardening đặc thù của công ty
- cần thêm package/nội dung nội bộ
- cần kiểm soát cực chặt phiên bản và cấu hình
- cần tối ưu image theo chuẩn bảo mật/nội bộ

<span style="color:red">Tóm lại: Bitnami rất tiện để học, thử nghiệm và triển khai nhanh, nhưng trong môi trường production nghiêm ngặt thì vẫn cần cân nhắc yêu cầu bảo mật và chuẩn nội bộ.</span>

## Kết luận

==Trong ngữ cảnh bài này, `Bitnami image` là image Docker đã được chuẩn bị sẵn cho PostgreSQL cluster, kèm script và cơ chế cấu hình giúp việc deploy `StatefulSet` dễ hơn.==

Vì vậy người học có thể tập trung vào các khái niệm Kubernetes quan trọng như ==persistence, networking, replication và high availability==.
