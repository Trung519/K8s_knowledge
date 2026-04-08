---
doc_id: kubernetes-architecture-explained
title: Kiến trúc Kubernetes từ control plane đến worker node
legacy_title: Kubernetes Architecture Explained (2026 Updated Edition)
type: concept-note
domain: kubernetes
keywords:
  - kubernetes
  - architecture
  - control-plane
  - worker-node
  - kube-apiserver
  - etcd
  - kube-scheduler
  - kubelet
  - kube-proxy
  - cni
  - container-runtime
status: active
---

# Kiến trúc Kubernetes từ control plane đến worker node

Nguồn tham khảo:
- [DevOpsCube - Kubernetes Architecture Explained (2026 Updated Edition)](https://devopscube.com/kubernetes-architecture-explained/)

Metadata của bài gốc:
- Tác giả: `Bibin Wilson`
- Ngày publish trên trang: `2026-01-19`
- Ngày chỉnh sửa trên trang: `2026-03-26`

## Kết luận ngắn gọn trước

==Kubernetes là một distributed system gồm nhiều thành phần chạy trên nhiều server khác nhau trong cùng một cluster.==

==Cluster Kubernetes luôn có 2 khối chính: `control plane` và `worker node`.==

==`control plane` quyết định cluster nên ở trạng thái nào; `worker node` là nơi thật sự chạy workload.==

==`kube-apiserver` là cổng vào trung tâm, `etcd` là nơi lưu state, `scheduler` chọn node, `controller manager` liên tục kéo cluster về desired state, còn `kubelet` + `container runtime` là cặp thực thi pod trên node.==

## Sơ đồ tổng quan

![Kubernetes Architecture Overview](assets/kubernetes-architecture-explained/overview.png)

![Kubernetes Architecture Diagram](assets/kubernetes-architecture-explained/cluster-diagram.gif)

## Hiểu đúng "Kubernetes architecture" là gì

Theo bài viết, khi nói về kiến trúc Kubernetes là đang nói đến:

- các thành phần lõi của cluster
- vai trò của từng thành phần
- cách các thành phần này kết nối với nhau
- cách các hệ thống bên ngoài đi vào cluster

Điểm rất quan trọng cần nhớ:

- ==Kubernetes không phải một tiến trình đơn lẻ.==
- ==Nó là một hệ phân tán có nhiều process/component phối hợp với nhau qua mạng.==
- server trong cluster có thể là VM hoặc bare metal

## Hai lớp lớn của cluster

### 1. Control plane

`control plane` chịu trách nhiệm:

- orchestration container
- duy trì `desired state` của cluster
- điều phối toàn bộ hành vi của cluster

Các thành phần chính:

1. `kube-apiserver`
2. `etcd`
3. `kube-scheduler`
4. `kube-controller-manager`
5. `cloud-controller-manager`

Lưu ý:

- ==Cluster có thể có một hoặc nhiều control plane node.==

### 2. Worker node

`worker node` chịu trách nhiệm:

- chạy ứng dụng containerized
- thực thi các pod đã được điều phối

Các thành phần chính:

1. `kubelet`
2. `kube-proxy`
3. `container runtime`

## 1. kube-apiserver

![kube-apiserver Diagram](assets/kubernetes-architecture-explained/apiserver.gif)

`kube-apiserver` là trung tâm giao tiếp của cluster:

- expose Kubernetes API
- nhận request từ người dùng, tool, controller, kubelet và các thành phần khác
- là điểm vào gần như bắt buộc cho mọi tương tác với cluster

Khi dùng `kubectl`, thực chất ở backend là:

- `kubectl` gọi HTTP REST API
- kết nối tới API server
- giao tiếp qua TLS

Những gì cần nhớ về API server:

1. ==Đây là "central hub" của cluster.==
2. expose API endpoint và xử lý request tới cluster
3. API có version và có thể hỗ trợ nhiều version đồng thời
4. thực hiện authentication và authorization
5. xử lý admission như validation và mutation cho object
6. điều phối luồng thông tin giữa control plane và worker node
7. có `aggregation layer` để mở rộng API bằng custom API resources và controller
8. hỗ trợ `watch` resource để client và các component nhận thay đổi theo thời gian thực
9. từng component như kubelet, scheduler, controller tự watch API server để biết việc cần làm

Một ý cực quan trọng trong bài:

- ==`kube-apiserver` chỉ chủ động initiate connection tới `etcd`.==
- ==Các component còn lại chủ động kết nối tới API server.==

Bài cũng nhắc đến:

- `apiserver proxy` là một phần trong tiến trình API server
- nó có thể được dùng để hỗ trợ truy cập `ClusterIP service` từ bên ngoài cluster

### Ghi nhớ về bảo mật API server

==API server phải được bảo vệ cực kỹ vì đây là cửa ngõ của cluster.==

Bài viết dẫn lại một thống kê của Shadowserver Foundation rằng đã từng phát hiện `380,000` API server Kubernetes public trên Internet. Ý cần nhớ ở đây không phải con số tuyệt đối, mà là:

- expose nhầm API server ra Internet là rủi ro cực lớn
- phải siết `TLS`, authn, authz, network access và RBAC

## 2. etcd

![etcd Diagram](assets/kubernetes-architecture-explained/etcd.gif)

`etcd` là nơi lưu dữ liệu cốt lõi của cluster.

Theo cách diễn đạt của bài:

- nó vừa là backend service discovery
- vừa là distributed database
- có thể xem như "brain of the Kubernetes cluster"

### etcd là gì

`etcd` là:

- open-source
- strongly consistent
- distributed
- key-value store

Giải thích nhanh:

- `strongly consistent`: update phải được phản ánh nhất quán trên cluster
- `distributed`: chạy được trên nhiều node
- `key-value store`: dữ liệu lưu theo cặp key-value

Bài viết nói datastore của etcd được xây trên `bbolt`.

### Cơ chế đồng thuận

==`etcd` dùng thuật toán `Raft` để đạt consistency và availability.==

Mô hình hoạt động:

- leader
- member/follower

Mục tiêu:

- tăng HA
- chịu được node failure

### etcd làm gì cho Kubernetes

Những ý cần nhớ:

1. ==Toàn bộ config, state và metadata của object Kubernetes được lưu trong etcd.==
2. object ví dụ: `pods`, `secrets`, `daemonsets`, `deployments`, `configmaps`, `statefulsets`
3. API server dùng khả năng `Watch()` của etcd để theo dõi thay đổi trạng thái object
4. etcd expose key-value API qua `gRPC`
5. dữ liệu object được lưu dưới nhánh `/registry`

Ví dụ mà bài đưa ra:

```text
/registry/pods/default/nginx
```

Đó là vị trí lưu thông tin của pod `nginx` trong namespace `default`.

### Điểm cần nhớ rất quan trọng

- ==`etcd` là thành phần stateful duy nhất trong control plane mà bài nhấn mạnh.==
- ==Nếu etcd hỏng, ứng dụng đang chạy có thể vẫn tiếp tục chạy, nhưng cluster sẽ không thể tạo hoặc update object mới một cách bình thường.==

### Fault tolerance của etcd
Fault tolerance: là khả năng hoạt động của 1 service nào đó ngay cả khi bị lỗi.

| Số node etcd | Chịu lỗi được | Quorum |
|---|---|---|
| 3 | 1 node | 2 |
| 5 | 2 node | 3 |
| 7 | 3 node | 4 |
Bảng này nói rằng nếu có 3 node etcd thì 1 node dead -> vẫn work bình thường. nhưng từ 2 node dead -> dead

Công thức bài viết đưa ra:

```text
fault tolerance = (n - 1) / 2
```

Đây là cơ chế: quorum: đa số: tức là muốn cluster hoạt động ổn định phải có hơn 1/2 no working được.
Trong đó `n` là tổng số node etcd.

## 3. kube-scheduler

![Scheduler Overview](assets/kubernetes-architecture-explained/scheduler-overview.gif)

![Scheduler Logic](assets/kubernetes-architecture-explained/scheduler-logic.gif)

`kube-scheduler` chịu trách nhiệm chọn worker node phù hợp cho pod.

Khi deploy pod, pod có thể mang theo các ràng buộc như:

- CPU
- memory
- affinity
- taints / tolerations
- priority
- persistent volume

==Việc của scheduler là nhìn vào request tạo pod, rồi chọn node phù hợp nhất để pod đó chạy.==

### Scheduler chọn node như thế nào

Theo bài, scheduler hoạt động theo 2 pha lớn:

1. `filtering`
2. `scoring`

#### Filtering

- tìm ra danh sách node đủ điều kiện để chạy pod
- nếu không có node phù hợp thì pod trở thành `unschedulable`
- pod sẽ quay lại scheduling queue

Một ý đáng nhớ:

- scheduler không nhất thiết phải scan toàn bộ node trong cluster lớn
- có tham số `percentageOfNodesToScore`
- mặc định thường là `50%`
- với cluster rất lớn, mặc định có thể là `5%`

#### Scoring

- các node đã qua filter sẽ được chấm điểm
- scheduler dùng nhiều plugin để score
- node có điểm cao nhất sẽ được chọn
- nếu điểm ngang nhau, scheduler có thể chọn ngẫu nhiên

Sau khi chọn được node:

- scheduler tạo binding event trong API server
- nghĩa là gắn pod với node

### Những điều quan trọng về scheduler

1. lắng nghe sự kiện tạo pod từ API server
2. có `Scheduling cycle` và `Binding cycle`
	1. Scheduling cycle: quá trình tìm filter + score để tìm ra node phù hợp để đặt pod lên
	2. Binding cycle: Sau khi chọn được thì start pod trên node đó.
3. pod priority cao sẽ được ưu tiên scheduling trước
4. có thể có custom scheduler song song với scheduler mặc định
5. có scheduling framework kiểu plugin nên có thể custom workflow

### Điểm mới mà bài cập nhật cho 2026

Bài nhắc tới `Dynamic Resource Allocation (DRA)`:

- stable từ `Kubernetes v1.34`
- cho scheduler biết thông tin phần cứng chuyên biệt tốt hơn
- hữu ích với `GPU`, `FPGA`, `smart NIC`
- đặc biệt phù hợp AI/ML workload

## 4. kube-controller-manager

![Controller Manager Diagram](assets/kubernetes-architecture-explained/controller-manager.gif)

Muốn hiểu `kube-controller-manager` phải hiểu `controller` là gì.

Theo bài:

- controller là chương trình chạy `infinite control loops`
- luôn quan sát `actual state` và `desired state`
- nếu hai trạng thái lệch nhau thì controller tìm cách kéo actual về desired

Ví dụ rất dễ hiểu:

- manifest khai báo `Deployment` cần `2 replicas`
- nếu ai đó update lên `5 replicas`
- deployment controller nhận ra desired state mới là `5`
- controller đảm bảo cluster tiến về trạng thái đó

### Vai trò của kube-controller-manager

`kube-controller-manager` là component quản lý toàn bộ các controller built-in của Kubernetes.

Những controller được bài liệt kê:

1. `Deployment controller`
2. `ReplicaSet controller`
3. `DaemonSet controller`
4. `Job controller`
5. `CronJob controller`
6. `Endpoints controller`
7. `Namespace controller`
8. `ServiceAccounts controller`
9. `Node controller`

### Ý cốt lõi cần nhớ

- ==Controller manager không trực tiếp "chạy pod", mà nó liên tục reconcile trạng thái cluster.==
- ==Kubernetes mạnh ở chỗ nó vận hành theo mô hình desired state + reconcile loop.==

Bài cũng nhấn mạnh:

- có thể mở rộng Kubernetes bằng `custom controller`
- thường custom controller sẽ đi cùng `Custom Resource Definition (CRD)`

## 5. cloud-controller-manager (CCM)

![Cloud Controller Manager Diagram](assets/kubernetes-architecture-explained/cloud-controller-manager.gif)

`cloud-controller-manager` là cầu nối giữa:

- Kubernetes
- cloud provider API

Ý nghĩa lớn nhất:

- tách core Kubernetes ra khỏi chi tiết triển khai của từng cloud
- cho phép cloud provider tích hợp qua cloud controller binaries riêng

### CCM quản lý gì

Theo bài, CCM có các controller quan trọng sau:

1. `Node controller`
2. `Route controller`
3. `Service controller`

### Nhiệm vụ cụ thể

#### Node controller

- cập nhật thông tin node từ cloud API
- ví dụ hostname, CPU, memory, health, label, annotation

#### Route controller

- cấu hình route mạng trên cloud
- để pod ở các node khác nhau có thể giao tiếp

#### Service controller

- tạo load balancer cho service
- gán IP và tích hợp service với tài nguyên cloud

### Ví dụ thực tế mà bài nêu

1. tạo `Service type LoadBalancer` thì Kubernetes có thể provision cloud load balancer
2. provision storage volume cho pod dựa trên cloud storage

==Nhớ ngắn gọn: CCM quản lý lifecycle của tài nguyên cloud mà cluster Kubernetes dùng tới.==

## Worker node components

## 6. kubelet

![Kubelet Diagram](assets/kubernetes-architecture-explained/kubelet.png)

`kubelet` là agent chạy trên mọi node trong cluster.

Theo bài:

- nó không chạy dưới dạng container
- nó chạy như daemon do `systemd` quản lý

### kubelet làm gì

1. đăng ký worker node với API server
2. đọc `podSpec`
3. tạo, sửa, xoá container để đưa pod về desired state
4. xử lý `liveness`, `readiness`, `startup probes`
5. mount volume dựa trên cấu hình pod
6. báo cáo trạng thái node và pod lên API server

==Có thể hiểu kubelet là người "thi hành mệnh lệnh" trên từng node.==

### kubelet nhận podSpec từ đâu

Bài nhắc 4 nguồn:

- API server
- file
- HTTP endpoint
- HTTP server

Ví dụ quan trọng nhất là `static pod`.

### Static pod

`static pod`:

- do kubelet quản lý trực tiếp
- không do API server quản lý theo nghĩa thông thường

Use case rất quan trọng:

- khi bootstrap control plane
- kubelet đọc manifest tại:

```text
/etc/kubernetes/manifests
```

- từ đó khởi động `api-server`, `scheduler`, `controller-manager` dạng static pod

### Những điểm kỹ thuật cần nhớ về kubelet

1. dùng `CRI` qua gRPC để nói chuyện với container runtime
2. có HTTP endpoint để stream logs và mở exec sessions
3. dùng `CSI` qua gRPC cho block volume
4. dùng plugin `CNI` để cấp pod IP, route mạng và firewall rules

### Điểm mới mà bài nhắc

Theo bài, từ `Kubernetes v1.35` ở mức `General Availability`:

- kubelet có thể resize CPU/memory request và limit của pod khi pod đang chạy
- nhiều trường hợp không cần restart container
- đây là tính năng `in-place pod resize`

## 7. kube-proxy (optional)

![Kube Proxy Diagram](assets/kubernetes-architecture-explained/kube-proxy.png)

Muốn hiểu `kube-proxy` phải hiểu sơ bộ:

- `Service`
- `Endpoint`

### Nhắc nhanh về Service và Endpoint

`Service`:

- là cách expose một nhóm pod
- khi tạo service sẽ có `ClusterIP`
- `ClusterIP` chỉ truy cập được trong cluster

`Endpoint`:

- chứa danh sách IP và port của nhóm pod phía sau service

Bài nhắc một chi tiết dễ quên:

- ==không ping được `ClusterIP` như ping pod IP==
- vì `ClusterIP` dùng cho service discovery/routing, không phải kiểu IP "host thật"

### kube-proxy làm gì

`kube-proxy`:

- chạy trên mọi node dưới dạng `DaemonSet`
- là thành phần hiện thực khái niệm `Service`
- tạo rule mạng để chuyển traffic từ service sang backend pods
- xử lý load balancing ở mức network

Nó chủ yếu proxy:

- `TCP`
- `UDP`
- `SCTP`

Và:

- ==không hiểu HTTP ở tầng ứng dụng==

### Cơ chế hoạt động

1. kube-proxy hỏi API server để biết `Service`, `Endpoint`, IP và port
2. theo dõi thay đổi của service và endpoint
3. tạo hoặc cập nhật rule routing trên node

### Các mode mà bài liệt kê

1. `IPTables`
2. `NFTables`
3. `Userspace` (legacy, không khuyến nghị)
4. `Kernelspace` (Windows)

Ghi chú quan trọng từ bài:

- `IPTables` được mô tả là mode mặc định
- khi kết nối đã được thiết lập thì traffic sẽ tiếp tục vào cùng backend pod cho đến khi connection đóng
- `NFTables` nhắm tới cải thiện giới hạn hiệu năng/scalability của `iptables`, đặc biệt trong cluster lớn

### Vì sao kube-proxy được ghi là "optional"

==Bài nhấn mạnh rằng kube-proxy không còn là bắt buộc trong mọi cluster hiện đại.==

Lý do:

- một số CNI hiện đại đã tự triển khai packet forwarding và load balancing
- khi đó cluster vẫn networking bình thường mà không cần kube-proxy

Ví dụ điển hình:

- `Cilium`
- dùng `eBPF`
- xử lý Service traffic trực tiếp trong kernel Linux

## 8. Container runtime

![Container Runtime Diagram](assets/kubernetes-architecture-explained/container-runtime.png)

`container runtime` là phần mềm thật sự chạy container trên node.

Trách nhiệm chính:

- pull image từ registry
- chạy container
- cấp phát và cô lập resource
- quản lý lifecycle của container

### Hai khái niệm phải nhớ

#### CRI

`Container Runtime Interface` là:

- bộ API cho phép Kubernetes giao tiếp với runtime
- giúp nhiều runtime có thể thay thế cho nhau
- định nghĩa cách tạo, start, stop, delete container
- quản lý image và networking liên quan container

#### OCI

`Open Container Initiative` là:

- bộ chuẩn cho format container và runtime

### Mối quan hệ giữa kubelet và runtime

- kubelet giao tiếp với runtime thông qua `CRI APIs`
- runtime đảm nhiệm phần triển khai container ở node
- kubelet lấy thông tin container từ runtime rồi phản ánh ngược lên control plane

### Ví dụ workflow với CRI-O

Theo bài, luồng mức cao là:

1. API server có request tạo pod mới
2. kubelet nói chuyện với `CRI-O`
3. `CRI-O` kiểm tra và pull image cần thiết
4. `CRI-O` sinh `OCI runtime specification`
5. `CRI-O` gọi runtime tương thích OCI như `runc` để start process container

==Nhớ ngắn gọn: kubelet ra lệnh, container runtime thi công.==

## Addon components

Bài viết nói rõ:

- chỉ core components thì cluster chưa đủ "fully operational"
- cần thêm addon tùy use case

Các addon phổ biến mà bài liệt kê:

1. `CNI Plugin`
2. `CoreDNS`
3. `Metrics Server`
4. `Kubernetes Dashboard`

### CoreDNS

Vai trò:

- DNS server bên trong cluster
- bật service discovery dựa trên DNS

### Metrics Server

Vai trò:

- thu thập resource metrics của node và pod
- phục vụ quan sát và autoscaling ở mức cơ bản

### Kubernetes Dashboard

Vai trò:

- giao diện web để quản lý object

## 9. CNI Plugin

![CNI Plugin Diagram](assets/kubernetes-architecture-explained/cni-plugin.png)

`CNI` là viết tắt của `Container Networking Interface`.

Theo bài:

- đây là kiến trúc plugin
- có specification và thư viện trung lập nhà cung cấp
- dùng để tạo network interface cho container

Điểm cần nhớ:

- ==CNI không chỉ dành riêng cho Kubernetes.==
- nó có thể được dùng rộng hơn trong hệ sinh thái container/orchestrator

### Vì sao cần CNI plugin

Mỗi tổ chức có nhu cầu mạng khác nhau:

- isolation
- security
- encryption
- routing model

Vì vậy có nhiều CNI plugin từ nhiều nhà cung cấp khác nhau để chọn.

### CNI làm việc với Kubernetes thế nào

Theo bài:

1. `kube-controller-manager` gán `pod CIDR` cho từng node
2. kubelet kết hợp container runtime để khởi chạy pod
3. CRI plugin trong runtime gọi sang CNI plugin để cấu hình mạng pod
4. CNI plugin giúp pod giao tiếp được dù nằm trên cùng node hay khác node

### Chức năng nổi bật

1. pod networking
2. network security và isolation bằng `NetworkPolicy`

### Ví dụ CNI plugin mà bài nhắc

1. `Calico`
2. `Flannel`
3. `Cilium`
4. `Amazon VPC CNI`
5. `Azure CNI`

## Kubernetes native objects mà kiến trúc này quản lý

Sau khi hiểu các thành phần, bài gom lại thành các nhóm object mà Kubernetes architecture quản lý.

### Workload objects

1. `Pod`
2. `ReplicaSet`
3. `Deployment`
4. `DaemonSet`
5. `StatefulSet`
6. `Job`
7. `CronJob`

### Configuration và secrets

1. `ConfigMap`
2. `Secret`

### Networking objects

1. `Service`
2. `Ingress`
3. `Gateway API`
4. `NetworkPolicy`

### Mở rộng Kubernetes

1. `Custom Resource Definitions (CRDs)`
2. `Custom Controllers / Operators`

==Điểm quan trọng: Kubernetes không chỉ quản lý object built-in. Nó còn quản lý được object mở rộng thông qua CRD + controller/operator.==

### Ghi chú AI/ML trong bài

Bài có thêm một ý cập nhật cho bối cảnh mới:

- Kubernetes đang có thêm native features cho AI/ML
- ví dụ: `Device Plugins`, `Gateway API Inference`, `OCI image volumes`

## Workflow end-to-end của cluster

Phần này là bản tổng hợp lại từ toàn bộ bài để dễ hình dung luồng vận hành đầu-cuối.

1. Người dùng hoặc hệ thống gọi `kubectl` / API client tới `kube-apiserver` qua `TLS`.
2. API server xác thực, phân quyền, chạy admission và ghi object state vào `etcd`.
3. `controller manager` watch API server rồi reconcile các object về desired state.
4. Nếu pod chưa có node, `scheduler` lấy pod đó, filter + score node, rồi bind pod vào một worker node.
5. `kubelet` trên node đích đọc `podSpec` từ API server.
6. `kubelet` gọi `container runtime` qua `CRI` để pull image và start container.
7. runtime và `CNI plugin` cấu hình mạng cho pod.
8. `kube-proxy` hoặc logic tương đương từ CNI xử lý service routing/load balancing.
9. `kubelet` tiếp tục báo cáo health, status, probe result về API server.

==Cả hệ thống hoạt động bằng cơ chế watch + reconcile + desired state, chứ không phải một tiến trình duy nhất chạy tuần tự từ đầu đến cuối.==

## Những điểm dễ nhầm nhưng phải nhớ

### `kube-apiserver` không phải nơi lưu dữ liệu cuối cùng

- API server là cửa vào và bộ điều phối
- `etcd` mới là nơi lưu trạng thái cluster

### `scheduler` không chạy container

- scheduler chỉ chọn node và bind pod
- `kubelet` + `runtime` mới là cặp chạy container thật sự

### `controller manager` không phải scheduler

- controller manager reconcile desired state của object
- scheduler chỉ giải bài toán placement

### `kube-proxy` không phải ingress controller

- kube-proxy làm routing/load balancing ở mức service network
- không xử lý HTTP routing nâng cao kiểu host/path như ingress hoặc gateway

### `CNI` không thay `container runtime`

- CNI lo networking
- runtime lo vòng đời container

## FAQ quan trọng của bài

### Control plane dùng để làm gì

==Duy trì desired state của cluster và ứng dụng đang chạy trên cluster.==

### Worker node dùng để làm gì

- là nơi container thật sự chạy
- nhận chỉ thị từ control plane

### Giao tiếp giữa control plane và worker node được bảo vệ thế nào

- dùng `PKI certificates`
- dùng `TLS` giữa các thành phần

### Nếu etcd down thì chuyện gì xảy ra

- workload đang chạy có thể vẫn tiếp tục chạy
- nhưng cluster không thể tạo hoặc cập nhật object mới đúng cách nếu etcd chưa hoạt động lại

### kube-proxy còn bắt buộc không

- không hẳn
- cluster hiệu năng cao ngày nay có thể dùng `eBPF` networking như `Cilium`

### Kubernetes là PaaS hay IaaS

Theo bài:

- nó có đặc tính của cả `PaaS` và `IaaS`

### L4 và L7 routing trong Kubernetes là gì

- `Service` dùng IP/Port nên gần với `L4 routing`
- `Gateway API` dùng host/path/HTTP metadata nên là `L7 routing`

## Ý chốt của bài

==Hiểu kiến trúc Kubernetes sẽ giúp triển khai, vận hành và troubleshooting cluster tốt hơn trong công việc hằng ngày.==

==Nền tảng của Kubernetes không chỉ là "chạy container", mà là quản trị một distributed system dựa trên state, controller loop và các interface chuẩn như CRI, CNI, CSI.==

==Theo góc nhìn của bài viết, Kubernetes trong bối cảnh 2026 đang tiến gần hơn tới vai trò "distributed operating system", đặc biệt với các tính năng như in-place scaling, GPU-centric scheduling và eBPF-powered networking.==

## Bản học thuộc siêu nhanh

1. `kube-apiserver` là cổng vào trung tâm của cluster.
2. `etcd` lưu state và metadata của cluster.
3. `scheduler` chọn node cho pod.
4. `controller manager` liên tục kéo cluster về desired state.
5. `kubelet` hiện thực podSpec trên worker node.
6. `container runtime` thật sự chạy container.
7. `kube-proxy` hiện thực Service networking nhưng có thể optional.
8. `CNI plugin` lo networking giữa các pod.
9. `TLS` và `PKI` là nền tảng bảo mật giao tiếp nội bộ cluster.
10. Toàn bộ Kubernetes vận hành theo mô hình distributed system + watch + reconcile.
