---
doc_id: raft-etcd-quorum-vs-repmgr
title: RAFT, etcd quorum và khác biệt với repmgr
legacy_title: RAFT etcd quorum vs repmgr
type: concept-note
domain: kubernetes
keywords:
  - raft
  - etcd
  - quorum
  - leader-election
  - consensus
  - repmgr
  - postgresql
status: active
---

# RAFT, etcd quorum và khác biệt với repmgr

Nguồn ghi chú:
- Tổng hợp từ phần giải thích nội bộ về `RAFT`, `etcd quorum` và so sánh với `repmgr`

## Kết luận ngắn gọn trước

==`RAFT` là thuật toán đồng thuận giúp nhiều node giữ cùng dữ liệu và cùng thứ tự ghi.==

==Cốt lõi của RAFT là: `1 leader` điều phối, các node cùng giữ data, và chỉ `commit` khi đa số node đồng ý.==

==`repmgr` không phải là RAFT. `repmgr` là tool HA cho PostgreSQL; còn `RAFT` là cơ chế đồng thuận ở lõi hệ phân tán như `etcd`.==

## 1) RAFT là gì?

`RAFT` là một **consensus algorithm**.

Mục tiêu của nó:

- giúp nhiều node trong một cluster thống nhất với nhau
- đảm bảo dữ liệu được ghi theo **cùng một thứ tự**
- giảm rủi ro mỗi node hiểu state khác nhau

Nếu nói ngắn gọn:

- ==RAFT giải bài toán: "nhiều máy cùng giữ một state, vậy ai được quyền ghi và khi nào bản ghi đó được xem là hợp lệ?"==

## 2) 3 ý cốt lõi của RAFT

### 1. Leader election

Cluster sẽ bầu ra một `leader`.

Leader là node:

- nhận ghi mới
- điều phối việc replicate log
- đại diện cluster trong các thao tác ghi

Nếu leader chết:

- cluster sẽ bầu leader mới

### 2. Log replication

Khi có ghi mới:

- leader ghi entry mới vào log của chính nó
- leader gửi entry đó tới các follower
- follower append entry theo đúng thứ tự leader gửi

Ý quan trọng:

- ==RAFT không chỉ cần "có cùng dữ liệu", mà còn cần "cùng thứ tự ghi".==

### 3. Quorum

Một entry chỉ được xem là `commit` khi:

- leader nhận được xác nhận từ **đa số node**

Đây là phần quan trọng nhất để nhớ.

==Không cần tất cả node cùng phản hồi, nhưng bắt buộc phải có đa số.==

## 3) Quorum là gì?

`quorum` nghĩa là **đa số**.

Trong cluster `n` node:

- cần **hơn một nửa** số node đang sống và đồng ý

Ví dụ:

| Tổng số node | Quorum cần có |
|---|---|
| 1 | 1 |
| 3 | 2 |
| 5 | 3 |
| 7 | 4 |

Nhìn nhanh sẽ thấy:

- 3 node thì cần 2
- 5 node thì cần 3
- 7 node thì cần 4

==Quorum là lý do vì sao cluster vẫn chạy được khi mất một vài node, nhưng sẽ dừng ghi nếu mất quá nhiều node.==

## 4) Công thức fault tolerance là gì?

```text
fault tolerance = (n - 1) / 2
```

Trong đó:

- `n` = số node trong cluster đồng thuận

Với ngữ cảnh `etcd`:

- ==`n` là số `etcd member` trong `etcd cluster`==
- ==không phải số worker node Kubernetes==

### Ví dụ

#### Cluster etcd 3 node

- `n = 3`
- fault tolerance = `(3 - 1) / 2 = 1`

Nghĩa là:

- mất 1 node vẫn chạy
- mất 2 node thì không còn quorum

#### Cluster etcd 5 node

- `n = 5`
- fault tolerance = `(5 - 1) / 2 = 2`

Nghĩa là:

- mất 2 node vẫn chạy
- mất 3 node thì không còn quorum

## 5) Leader có lưu data không?

👉 ==Có.==

Đây là chỗ rất dễ hiểu nhầm.

Leader trong RAFT:

- vừa điều phối
- vừa lưu dữ liệu
- vừa tham gia quorum

Nó **không phải** kiểu:

- một con chỉ đứng ngoài điều khiển
- còn data nằm ở các node khác

Hiểu đúng phải là:

- ==leader cũng là một member thật trong cluster==
- ==leader cũng giữ log/data như follower==

### Ví dụ dễ nhớ

Giả sử etcd cluster có 3 node:

- `etcd-1` = leader
- `etcd-2` = follower
- `etcd-3` = follower

Khi có lệnh tạo object mới trong Kubernetes:

1. `kube-apiserver` ghi vào leader `etcd-1`
2. `etcd-1` append log mới
3. `etcd-1` gửi log sang `etcd-2` và `etcd-3`
4. nếu ít nhất 1 follower xác nhận, tổng cộng đã có 2/3 node đồng ý
5. entry được commit

Nghĩa là:

- leader có data
- follower cũng có data
- quorum xác nhận entry đó hợp lệ

## 6) RAFT hoạt động thế nào theo kiểu "nhớ là dùng được ngay"

Hãy tưởng tượng cluster có 3 node:

- A = leader
- B = follower
- C = follower

Ứng dụng gửi một lệnh ghi mới.

### Trường hợp bình thường

1. Leader A nhận lệnh ghi.
2. A ghi log vào local của A.
3. A replicate log sang B và C.
4. B xác nhận, C có thể chậm hơn.
5. Vì đã có A + B = 2/3, entry được commit.

Điểm đáng nhớ:

- ==không cần chờ 3/3==
- ==chỉ cần đủ quorum==

### Trường hợp một follower chết

Giả sử C chết:

1. A vẫn nhận ghi.
2. A replicate sang B.
3. A + B = 2/3.
4. Cluster vẫn commit được.

### Trường hợp leader chết

Giả sử A chết:

1. B và C phát hiện leader mất.
2. Cluster tổ chức bầu leader mới.
3. Một node, ví dụ B, trở thành leader mới.
4. Cluster tiếp tục phục vụ nếu vẫn đủ quorum.

## 7) Tại sao RAFT chặt chẽ về consistency?

Vì RAFT không cho phép entry được xem là hợp lệ chỉ vì "một node nào đó đã ghi".

Nó yêu cầu:

- có leader hợp lệ
- log đi theo thứ tự
- commit chỉ xảy ra khi đa số xác nhận

==Điểm mạnh của RAFT là consistency rất rõ ràng và cơ chế failover gắn chặt với đồng thuận dữ liệu.==

## 8) RAFT và repmgr giống nhau ở đâu?

Nếu nhìn ở mức rất cao thì chúng có vài điểm giống:

- đều có khái niệm một node chính
- đều có failover khi node chính chết
- đều cần cơ chế phát hiện node nào đang giữ vai trò chính

Vì vậy mới dễ bị nhầm.

## 9) RAFT và repmgr khác nhau ở đâu?

Đây là phần quan trọng nhất.

| Tiêu chí | RAFT | repmgr |
|---|---|---|
| Bản chất | Thuật toán đồng thuận | Tool quản lý HA cho PostgreSQL |
| Bài toán chính | Đồng thuận dữ liệu/log giữa nhiều node | Quản lý replication, monitoring, switchover, failover PostgreSQL |
| Mức độ kiểm soát consistency | Rất chặt ở lõi thuật toán | Phụ thuộc vào cơ chế replication và trạng thái PostgreSQL |
| Vai trò leader/primary | Gắn liền với commit quorum | Gắn với role database primary/standby |
| Ngữ cảnh phổ biến | etcd, distributed systems | PostgreSQL HA cluster |

### Cách nhớ nhanh

- ==RAFT = luật chơi để cả cluster thống nhất dữ liệu==
- ==repmgr = công cụ vận hành PostgreSQL HA==

## 10) Vì sao không nên đồng nhất RAFT với repmgr?

Vì chúng giải **hai lớp bài toán khác nhau**.

### RAFT

Trả lời câu hỏi:

- ai là leader?
- log nào là hợp lệ?
- khi nào bản ghi được commit?
- làm sao nhiều node giữ cùng một state?

### repmgr

Trả lời câu hỏi:

- PostgreSQL node nào là primary?
- node nào là standby?
- failover/switchover thế nào?
- đăng ký cluster member, monitor và promote ra sao?

==repmgr không phải consensus algorithm giống RAFT.==

## 11) Liên hệ với etcd trong Kubernetes

Trong Kubernetes:

- `etcd` dùng `RAFT`
- nên `etcd` có leader
- các member còn lại là follower
- ghi vào etcd phải đi qua logic đồng thuận

Điều này giải thích vì sao:

- etcd rất phù hợp để giữ state cluster
- nhưng cũng rất nhạy với việc mất quorum

### Ví dụ với etcd 3 node

Giả sử có:

- `etcd-1`
- `etcd-2`
- `etcd-3`

Nếu:

- `etcd-3` down

thì:

- `etcd-1` và `etcd-2` vẫn đủ quorum
- cluster vẫn ghi state mới được

Nhưng nếu:

- chỉ còn lại `etcd-1`

thì:

- không còn quorum
- etcd không thể commit ghi mới một cách hợp lệ

## 12) Ví dụ cực đời thường để khóa kiến thức

Hãy tưởng tượng có 3 quản lý két sắt:

- A
- B
- C

Quy định là:

- một người làm trưởng ca = leader
- mọi lần ghi sổ két phải có đa số xác nhận

Nếu A là leader và muốn ghi một giao dịch mới:

1. A ghi vào sổ của A.
2. A gọi B và C xác nhận.
3. Chỉ cần A + B đồng ý là giao dịch được chốt.

Nếu chỉ còn mình A:

- A không được tự ý chốt giao dịch
- vì không còn đa số

Đó chính là tinh thần của quorum trong RAFT.

## 13) Những hiểu nhầm rất thường gặp

### “Leader chỉ điều phối chứ không giữ data”

Sai.

- leader vẫn giữ data
- leader vẫn là member thật của cluster

### “n trong công thức là số worker node”

Sai.

- trong ngữ cảnh etcd, `n` là số `etcd member`

### “Chỉ cần còn 1 node sống là cluster vẫn ghi được”

Sai.

- muốn ghi tiếp phải còn quorum

### “RAFT và repmgr là một”

Sai.

- chúng chỉ giống nhau ở bề mặt là có node chính và failover
- nhưng bản chất khác nhau

## 14) Chốt 1 câu dễ nhớ

> **RAFT = 1 leader điều phối + tất cả node cùng giữ dữ liệu + chỉ commit khi đa số đồng ý**

## 15) Bản nhớ siêu nhanh

1. `RAFT` là thuật toán đồng thuận.
2. Nó dựa trên `leader election`, `log replication`, `quorum`.
3. Leader có lưu data, không phải chỉ điều phối.
4. Chỉ khi đa số node đồng ý thì entry mới được commit.
5. `n` trong công thức fault tolerance là số node của cluster đồng thuận, với etcd là số `etcd member`.
6. `RAFT` khác `repmgr`: một bên là thuật toán đồng thuận, một bên là tool HA cho PostgreSQL.
