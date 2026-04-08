---
doc_id: pod-priority-preemption-pdb-qos
title: Pod Priority, Preemption, PDB và QoS trong Kubernetes
legacy_title: Pod Priority, Preemption, PDB, QoS
type: concept-note
domain: kubernetes
keywords:
  - pod-priority
  - priorityclass
  - preemption
  - pdb
  - poddisruptionbudget
  - qos
  - eviction
  - scheduler
status: active
---

# Pod Priority, Preemption, PDB và QoS trong Kubernetes

Nguồn tham khảo:
- [DevOpsCube - Kubernetes Pod Priority, PriorityClass, and Preemption Explained](https://devopscube.com/pod-priorityclass-preemption/)
- [Kubernetes Docs - Pod Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)
- [Kubernetes Docs - Pod Quality of Service Classes](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos)
- [Kubernetes Docs - Disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)

Metadata của bài gốc:
- Tác giả: `Bibin Wilson`
- Ngày publish trên trang: `2025-03-30`

## Kết luận ngắn gọn trước

==`Priority` trả lời câu hỏi: Pod nào quan trọng hơn khi scheduler phải chọn thứ tự xử lý.==

==`Preemption` là cơ chế scheduler đẩy Pod ưu tiên thấp ra khỏi node để nhường chỗ cho Pod ưu tiên cao đang pending.==

==`PDB` không dùng để xếp lịch hay gán ưu tiên; nó chỉ đặt ngân sách gián đoạn cho Pod của một ứng dụng, và trong preemption thì chỉ được tôn trọng theo kiểu "best effort".==

==`QoS` không quyết định preemption của scheduler; `QoS` chủ yếu liên quan tới eviction khi node bị thiếu tài nguyên.==

Điểm quan trọng nhất của nhóm khái niệm này là:

- ==`Priority/Preemption` thuộc vùng quyết định của `scheduler`.==
- ==`QoS` chủ yếu phát huy tác dụng trong `kubelet eviction` khi node pressure xảy ra.==
- ==`PDB` là lớp bảo vệ availability cho ứng dụng replicated, nhưng không phải "lá chắn tuyệt đối" trước Pod có priority rất cao.==

## 1. Pod Priority là gì?

`Pod Priority` là mức độ quan trọng của một Pod so với các Pod khác trong cluster.

Kubernetes dùng priority để:

- sắp thứ tự Pod trong hàng chờ scheduling
- quyết định Pod nào được ưu tiên xếp trước
- làm cơ sở cho cơ chế preemption khi tài nguyên node không đủ

Hiểu rất ngắn:

- số càng lớn -> Pod càng quan trọng
- Pod priority cao sẽ được scheduler xem xét trước Pod priority thấp

## 2. PriorityClass là gì?

Muốn gán priority cho Pod, ta không set trực tiếp một con số tùy ý trong Pod spec, mà thường dùng `PriorityClass`.

`PriorityClass` là object cấp cluster, không phải namespaced.

Nó ánh xạ:

- `tên class`
- sang `một giá trị priority dạng số nguyên`

Pod sẽ tham chiếu class đó qua `priorityClassName`.

Ví dụ:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-apps
value: 1000000
preemptionPolicy: PreemptLowerPriority
globalDefault: false
description: "Mission critical applications"
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  priorityClassName: high-priority-apps
  containers:
    - name: web
      image: nginx:latest
```

### Ý nghĩa các field cần nhớ

`value`
- số priority thực tế
- số càng lớn thì priority càng cao

`priorityClassName`
- tên class được Pod dùng để lấy ra priority

`preemptionPolicy`
- mặc định là `PreemptLowerPriority`
- nếu set `Never` thì Pod vẫn được xếp hàng với priority cao hơn, nhưng không được preempt Pod khác

`globalDefault`
- nếu `true`, Pod không khai báo `priorityClassName` sẽ lấy priority mặc định từ class này

## 3. Preemption là gì?

`Preemption` là cơ chế nhường chỗ cưỡng bức do scheduler thực hiện.

Nó xảy ra khi:

1. Có một Pod priority cao đang `Pending`.
2. Scheduler không tìm được node nào đủ tài nguyên để đặt Pod đó.
3. Scheduler thử tìm node mà nếu loại bỏ một số Pod priority thấp hơn thì Pod mới sẽ chạy được.
4. Các Pod priority thấp hơn bị chọn làm "victim" và bị evict để giải phóng tài nguyên.

Nói ngắn gọn:

- ==Preemption chỉ thật sự được kích hoạt khi có Pod priority cao đang chờ schedule mà cluster không còn chỗ phù hợp.==

## 4. Scheduler dùng Priority và Preemption như thế nào?

Luồng tư duy đúng là:

1. Pod mới vào scheduling queue.
2. Scheduler đọc `priorityClassName` để biết Pod này có priority bao nhiêu.
3. Pod priority cao được xét trước Pod priority thấp.
4. Nếu có node phù hợp thì bind bình thường.
5. Nếu không có node phù hợp thì mới xét tới preemption.

Điểm cần nhớ:

- ==Priority quyết định thứ tự xét trong queue.==
- ==Preemption là hành động "hy sinh Pod khác" để nhường chỗ, không phải lúc nào cũng xảy ra.==

## 5. PDB là gì?

`PDB` viết tắt của `PodDisruptionBudget`.

Nó dùng để nói với cluster rằng:

- ứng dụng replicated này cần giữ tối thiểu bao nhiêu Pod còn chạy
- hoặc cho phép tối đa bao nhiêu Pod bị unavailable cùng lúc

Ví dụ tư duy:

- ứng dụng có 3 replicas
- PDB yêu cầu `minAvailable: 2`
- nghĩa là tại mọi thời điểm nên còn ít nhất 2 Pod available

`PDB` rất hữu ích cho:

- application có quorum
- service cần giữ một mức HA tối thiểu
- drain node hoặc các voluntary disruption khác

Ví dụ:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: payment
```

## 6. QoS là gì?

`QoS` viết tắt của `Quality of Service`.

Kubernetes gán QoS cho Pod dựa trên cách ta khai báo:

- `requests`
- `limits`

Ba lớp QoS chính:

### `Guaranteed`

Mỗi container đều có:

- CPU request = CPU limit
- memory request = memory limit

Đây là lớp ít có khả năng bị evict nhất khi node bị thiếu tài nguyên.

### `Burstable`

Pod có request/limit nhưng không thỏa điều kiện `Guaranteed`.

Đây là lớp ở giữa.

### `BestEffort`

Không khai báo CPU/memory request và limit.

Đây là lớp dễ bị evict nhất khi node pressure xảy ra.

## 7. Mối liên hệ giữa Priority, Preemption, PDB và QoS

Đây là phần dễ nhầm nhất.

## Priority <-> Preemption

`Priority` là nguyên nhân, `Preemption` là hành động kéo theo trong một số tình huống.

- không có priority thì scheduler không có cơ sở rõ ràng để nói Pod nào quan trọng hơn
- khi Pod priority cao không schedule được, scheduler có thể preempt Pod priority thấp hơn

=> ==Priority là đầu vào, preemption là một chiến lược xử lý của scheduler.==

## Priority/Preemption <-> PDB

`PDB` không tạo priority và cũng không làm Pod "quan trọng hơn".

Nó chỉ nói:

- ứng dụng này không nên bị gián đoạn quá mức

Khi scheduler chọn victim để preempt:

- nó cố gắng tránh làm vi phạm PDB nếu có thể
- nhưng đây chỉ là `best effort`

Nghĩa là:

- ==PDB được scheduler cân nhắc khi preemption xảy ra==
- ==nhưng PDB không đảm bảo chặn được hoàn toàn preemption==

Nếu Pod priority rất cao cần được schedule và không còn cách nào khác:

- Pod priority thấp hơn vẫn có thể bị loại bỏ
- kể cả khi việc đó làm PDB bị vi phạm

Đây là ý cực quan trọng của bài gốc.

## Priority/Preemption <-> QoS

Hai cơ chế này liên quan nhưng không cùng vai trò.

`Scheduler preemption`:

- quan tâm chủ yếu tới `priority`
- không dùng `QoS` làm tiêu chí chính để chọn nạn nhân

`Kubelet eviction do node pressure`:

- xảy ra khi node thiếu tài nguyên thực tế như memory, disk, PID...
- lúc này `QoS` mới có vai trò mạnh

Nói ngắn:

- ==Preemption là bài toán của scheduler khi Pod mới không có chỗ chạy.==
- ==QoS là bài toán của kubelet khi node đang bị thiếu tài nguyên trong lúc Pod đã chạy rồi.==

## PDB <-> QoS

Hai thứ này gần như giải quyết hai lớp vấn đề khác nhau:

- `PDB` bảo vệ mức độ available của ứng dụng replicated trước disruption
- `QoS` ảnh hưởng thứ tự eviction khi node pressure xảy ra

`PDB` không thay thế `QoS`.
`QoS` cũng không thay thế `PDB`.

## 8. Phân biệt 3 cơ chế rất dễ bị trộn lẫn

### Cơ chế 1: Scheduling order

Ai được scheduler xét trước?

=> chủ yếu do `Priority`

### Cơ chế 2: Scheduler preemption

Ai bị đẩy ra để nhường chỗ cho Pod mới priority cao?

=> chủ yếu do `Priority`, có cân nhắc `PDB` theo kiểu best effort

### Cơ chế 3: Node pressure eviction

Khi node thiếu tài nguyên thật sự, Pod nào dễ bị đuổi trước?

=> `QoS` và priority đều có liên quan, nhưng đây là luồng của `kubelet`, không phải logic preemption của scheduler

## 9. Hiểu đúng câu "QoS liên quan tới Priority như thế nào?"

Bài DevOpsCube nhấn mạnh một điểm rất hay:

- kubelet xem xét `QoS` rồi tới `priority` trong bối cảnh eviction do resource shortage trên node
- nhưng scheduler preemption thì không dựa vào `QoS`

Tức là:

- ==`QoS` không quyết định việc Pod có preempt Pod khác hay không==
- ==`QoS` chủ yếu giúp dự đoán Pod nào dễ bị evict khi node pressure xảy ra==

Nếu muốn nhớ nhanh:

- `priority` = độ quan trọng chiến lược
- `qos` = mức đảm bảo tài nguyên

## 10. Các PriorityClass hệ thống mặc định

Kubernetes có sẵn hai class rất cao cho Pod hệ thống:

- `system-node-critical`
- `system-cluster-critical`

Ý nghĩa:

- bảo vệ các thành phần cực kỳ quan trọng của cluster
- giúp các Pod hệ thống này gần như luôn được xếp lịch trước workload thông thường

Ví dụ nhóm Pod thường gắn với class rất cao:

- `etcd`
- `kube-apiserver`
- `kube-scheduler`
- `controller-manager`
- `CoreDNS`
- các addon thiết yếu của cluster

## 11. Khi nào nên dùng PriorityClass?

Nên dùng khi trong cluster có phân tầng mức độ quan trọng của workload, ví dụ:

- logging agent
- metrics collector
- payment service
- ingress/controller cốt lõi
- DaemonSet hệ thống quan trọng

Ý tưởng đúng:

- không phải Pod nào cũng nên đẩy priority lên cao
- chỉ workload thật sự quan trọng mới nên dùng class cao

Nếu lạm dụng:

- nhiều Pod cùng priority cao sẽ làm cơ chế phân tầng mất ý nghĩa
- cluster dễ xảy ra preemption không mong muốn

## 12. Các hiểu lầm phổ biến

### Hiểu lầm 1: Có priority cao thì Pod chắc chắn luôn chạy được

Sai.

Pod vẫn phải thỏa:

- resource request
- affinity / anti-affinity
- taints / tolerations
- topology constraints
- các điều kiện scheduling khác

Nếu không có node nào phù hợp, Pod vẫn pending.

### Hiểu lầm 2: PDB sẽ chặn mọi kiểu Pod bị đẩy ra

Sai.

Trong preemption:

- scheduler cố gắng tôn trọng PDB
- nhưng không bảo đảm tuyệt đối

### Hiểu lầm 3: QoS càng cao thì scheduler càng ưu tiên schedule trước

Sai.

Scheduler không dùng QoS để quyết định preemption target theo cách nhiều người thường nghĩ.

### Hiểu lầm 4: Preemption và eviction là một

Sai.

Chúng liên quan nhưng khác bối cảnh:

- `preemption` = do scheduler kích hoạt để nhường chỗ cho Pod mới priority cao
- `eviction` = thường do kubelet hoặc Eviction API trong các tình huống disruption / node pressure

## 13. Sơ đồ tư duy ngắn gọn

```text
Pod mới tạo
   |
   v
Scheduler đọc priority từ PriorityClass
   |
   +--> Có node phù hợp -> bind Pod
   |
   +--> Không có node phù hợp
           |
           v
      Xét preemption
           |
           +--> Cố gắng chọn victim priority thấp hơn
           +--> Cố gắng tránh vi phạm PDB nếu có thể
           +--> Nếu cần, vẫn có thể vi phạm PDB
```

Và một luồng khác:

```text
Pod đang chạy trên node
   |
   v
Node bị thiếu tài nguyên
   |
   v
Kubelet eviction
   |
   +--> QoS có vai trò mạnh
   +--> Priority cũng ảnh hưởng thứ tự
```

## 14. Cách nhớ siêu nhanh

1. `PriorityClass` = cách gán mức ưu tiên cho Pod.
2. `Priority` = Pod nào quan trọng hơn trong scheduling.
3. `Preemption` = Pod quan trọng hơn có thể đẩy Pod ít quan trọng hơn ra khỏi node để lấy chỗ.
4. `PDB` = giới hạn mức gián đoạn chấp nhận được của một ứng dụng replicated.
5. `QoS` = mức chất lượng dịch vụ dựa trên request/limit, chủ yếu dùng trong eviction khi node pressure.

## 15. Ý chốt của cả cụm khái niệm

==Nếu nhìn từ góc scheduler, cặp quan trọng nhất là `Priority` và `Preemption`.==

==Nếu nhìn từ góc availability của ứng dụng, `PDB` là lớp bảo vệ mềm giúp hạn chế disruption, nhưng không phải tuyệt đối.==

==Nếu nhìn từ góc node bị cạn tài nguyên, `QoS` mới là tín hiệu quan trọng để dự đoán Pod nào dễ bị evict.==

## 16. Ghi nhớ thực chiến

- Muốn workload quan trọng được xét trước: dùng `PriorityClass`.
- Muốn hạn chế app replicated bị gián đoạn khi drain hay disruption: dùng `PDB`.
- Muốn kiểm soát hành vi khi node pressure: khai báo `requests/limits` rõ ràng để có `QoS` phù hợp.
- Đừng dùng `PriorityClass` như cách "buff" tất cả workload, vì cuối cùng cluster sẽ không còn phân tầng ưu tiên nữa.
