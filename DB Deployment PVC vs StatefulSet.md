# DB Deployment + PVC vs DB StatefulSet

Dưới đây là bản tổng hợp chi tiết để phân biệt rõ `Deployment + PVC` cho database và `StatefulSet` cho database, đồng thời nối nó với câu chuyện PostgreSQL HA, `primary/standby`, `preStop`, failover và các hiểu nhầm rất thường gặp.

---

# 1) Kết luận ngắn gọn trước

Nếu chỉ cần nhớ một ý, hãy nhớ thế này:

- `Deployment + PVC` phù hợp khi DB chạy **đơn giản**, thường là **single instance**, dev/test/lab, hoặc workload chưa cần identity cố định cho từng replica.
- `StatefulSet` phù hợp khi DB cần **mỗi pod có identity riêng**, **volume riêng**, **hostname ổn định**, và thường đi kèm **replication / clustering / failover**.

Kubernetes mô tả:

- `Deployment` thường dành cho workload "doesn't maintain state".  
  Reference: [Deployments | Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- `StatefulSet` dành cho ứng dụng cần:
  - stable, unique network identifiers
  - stable persistent storage
  - ordered, graceful deployment/scaling/rolling updates  
  Reference: [StatefulSets | Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)

**Câu chốt:**  
Không phải cứ là DB thì bắt buộc `StatefulSet`, và cũng không phải cứ là app thì luôn là `Deployment`.  
Thứ quyết định là: **replica đó có cần state/identity/storage ổn định hay không**.

---

# 2) DB chạy bằng `Deployment + PVC` là gì?

## Ý tưởng

Bạn vẫn dùng `Deployment` để quản lý pod DB, nhưng mount thêm `PersistentVolumeClaim` để dữ liệu không mất khi pod restart hoặc bị recreate.

## Nó giải quyết được gì?

- pod có thể bị thay thế
- dữ liệu vẫn nằm trên volume persistent
- phù hợp với DB **1 instance**

## Nó không giải quyết triệt để điều gì?

- không tạo ra identity ổn định kiểu `postgres-0`, `postgres-1`
- không tự tổ chức replication
- không phù hợp tự nhiên cho nhiều replica DB truyền thống cùng lúc

## Khi nào hợp?

- PostgreSQL / MySQL chạy một node
- môi trường dev
- môi trường test
- demo
- workload đơn giản, chưa cần HA thực sự

## Ví dụ rất thực tế

Một pod PostgreSQL:

- container `postgres`
- mount PVC vào `/var/lib/postgresql/data`
- pod chết thì Deployment tạo pod khác
- pod mới mount lại volume cũ

Kubernetes có ví dụ chính thức cho kiểu này với MySQL single-instance:

- [Run a Single-Instance Stateful Application](https://kubernetes.io/docs/tasks/run-application/run-single-instance-stateful-application)

Ngoài ra tutorial WordPress + MySQL cũng dùng MySQL với `Deployment` + persistent storage, nhưng tài liệu ghi rõ nó không dành cho production:

- [WordPress and MySQL with Persistent Volumes](https://kubernetes.io/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/)

## Hiểu đúng bản chất

`Deployment + PVC` cho DB nghĩa là:

- bạn đang giữ **data persistent**
- nhưng chưa hẳn đang có **clustered database identity**

Nói cách khác:

- có state ở tầng storage
- nhưng chưa có "stateful topology" mạnh ở tầng pod identity như `StatefulSet`

---

# 3) DB chạy bằng `StatefulSet` là gì?

## Ý tưởng

`StatefulSet` sinh ra cho các workload mà mỗi replica không interchangeable.

Kubernetes docs nói rất rõ:

- mỗi pod có sticky identity
- identity này giữ nguyên qua rescheduling
- StatefulSet hữu ích khi cần stable network identity, stable storage, ordered rollout  
  Reference: [StatefulSets | Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)

## Với DB thì điều này có ý nghĩa gì?

Ví dụ bạn có:

- `postgres-0`
- `postgres-1`
- `postgres-2`

thì:

- mỗi pod có tên ổn định
- mỗi pod có PVC riêng
- pod bị recreate vẫn quay lại đúng identity cũ
- rất phù hợp với replication / clustering / failover manager

## Khi nào hợp?

- PostgreSQL HA
- MySQL replication
- MongoDB replica set
- Cassandra
- Kafka
- Elasticsearch trong nhiều mô hình stateful

## Ví dụ chính thức

Kubernetes có tutorial cho MySQL replicated stateful app bằng `StatefulSet`:

- [Run a Replicated Stateful Application](https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application)

## Hiểu đúng bản chất

`StatefulSet` không tự biến DB thành HA, nhưng nó tạo nền cực hợp để DB HA vận hành:

- pod có identity riêng
- volume riêng
- hostname ổn định
- rollback / scale / restart có thứ tự hơn

---

# 4) So sánh trực diện: `Deployment + PVC` vs `StatefulSet` cho DB

| Tiêu chí | DB dùng Deployment + PVC | DB dùng StatefulSet |
|---|---|---|
| Tên pod | không ổn định theo kiểu ordinal | ổn định kiểu `db-0`, `db-1`, `db-2` |
| Replica có interchangeable không? | gần với tư duy interchangeable hơn, nhất là khi chỉ 1 pod | không, mỗi replica có identity riêng |
| Storage | có thể persistent qua PVC | persistent và thường tách riêng per-pod |
| PVC per replica | không tự nhiên cho topology nhiều replica | rất tự nhiên qua `volumeClaimTemplates` |
| Phù hợp single-instance DB | rất hợp | cũng dùng được nhưng đôi khi overkill |
| Phù hợp replication/cluster | không tự nhiên | rất phù hợp |
| Hostname ổn định cho replication | yếu hơn | là điểm mạnh cốt lõi |
| Ordered rollout / scale | không phải đặc điểm chính | có |
| Tình huống phổ biến | dev/test, DB đơn | production HA, cluster stateful |

**Câu chốt:**  
`Deployment + PVC` giải bài toán "pod thay thì data đừng mất".  
`StatefulSet` giải bài toán "mỗi DB node phải giữ đúng identity + storage + thứ tự vận hành".

---

# 5) Bức tranh lớn: đang có 2 tầng khác nhau

Bạn rất dễ bị rối vì đang trộn **tầng Kubernetes** và **tầng database HA**.

## Tầng 1: Kubernetes / StatefulSet

StatefulSet là workload controller dành cho ứng dụng cần:

- tên pod ổn định,
- storage ổn định,
- thứ tự tạo/xóa/rolling update có kiểm soát.

Kubernetes ghi rõ StatefulSet hữu ích khi cần **stable, unique network identifiers**, **stable persistent storage**, và **ordered, graceful deployment/scaling/rolling updates**.  
Reference: [StatefulSets | Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)

## Tầng 2: PostgreSQL HA

PostgreSQL HA là chuyện:

- ai là `primary`,
- ai là `standby`,
- dữ liệu replicate thế nào,
- khi primary chết thì failover ra sao.

PostgreSQL docs nói rõ server được phép ghi là **primary**, server bám theo thay đổi từ primary là **standby**; hot standby có thể nhận kết nối đọc, còn warm standby thì chưa nhận kết nối cho đến khi promote.  
Reference: [High Availability, Load Balancing, and Replication](https://www.postgresql.org/docs/current/high-availability.html)

**Câu chốt:**  
`StatefulSet` không tự làm replication DB. Nó chỉ tạo môi trường “ổn định” để database HA chạy đúng.

---

# 6) Primary và standby là gì?

## Primary

Là máy DB chính, nhận:

- `INSERT`
- `UPDATE`
- `DELETE`

## Standby

Là máy DB phụ, theo dõi thay đổi từ primary.  
Nếu là **hot standby**, nó có thể nhận **read-only queries**.

PostgreSQL docs nói hot standby cho phép kết nối và chạy truy vấn chỉ đọc; các kết nối đó là strictly read-only.  
Reference: [Hot Standby](https://www.postgresql.org/docs/current/hot-standby.html)

## Ví dụ

- `postgres-0` = primary
- `postgres-1` = standby
- `postgres-2` = standby

App ghi dữ liệu thì thường phải ghi vào **primary**.  
App đọc dữ liệu có thể đọc từ primary hoặc từ standby tùy kiến trúc.  
Reference: [High Availability, Load Balancing, and Replication](https://www.postgresql.org/docs/current/high-availability.html)

---

# 7) Vì sao DB không giống app pod phía sau Service?

Đây là hiểu nhầm phổ biến nhất.

## Với app stateless

Nếu bạn có 3 pod API giống nhau:

- pod A
- pod B
- pod C

thì request vào pod nào cũng gần như như nhau.

## Với PostgreSQL primary/standby

3 pod DB **không giống nhau về vai trò**:

- 1 thằng có quyền ghi,
- 2 thằng còn lại chỉ theo sau hoặc chỉ đọc.

PostgreSQL docs nói vấn đề cốt lõi của DB là **synchronization problem**: write vào server nào cũng phải được propagate để các server khác trả kết quả nhất quán. Vì thế DB read/write khó load-balance đơn giản như web server.  
Reference: [High Availability, Load Balancing, and Replication](https://www.postgresql.org/docs/current/high-availability.html)

**Câu chốt:**  
DB HA không phải kiểu “service tự chọn pod nhàn nhất rồi bắn request vào”. Với DB, vai trò node rất quan trọng.

---

# 8) StatefulSet thực sự làm gì?

StatefulSet:

- giữ tên pod ổn định như `postgres-0`, `postgres-1`, `postgres-2`,
- giúp pod mới quay lại với đúng identity cũ,
- giúp volume cũ gắn lại đúng pod khi pod được thay thế.

Kubernetes docs nói pod trong StatefulSet có **sticky identity**, không interchangeable, và identifier đó giữ nguyên qua các lần rescheduling; điều này giúp ghép volume cũ vào pod thay thế.  
Reference: [StatefulSets | Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)

## Ví dụ

`postgres-0` chết.  
StatefulSet sẽ cố tạo lại **đúng `postgres-0`**, chứ không tạo ngẫu nhiên một pod mới tên khác rồi coi như xong.

**Nhưng:** pod mới đó **không tự nhiên trở thành primary**.  
Vai trò primary/standby là do PostgreSQL HA/repmgr/failover manager quyết định.  
Reference: [Failover](https://www.postgresql.org/docs/current/warm-standby-failover.html)

---

# 9) Failover là gì?

PostgreSQL docs nói: nếu primary fail thì standby nên bắt đầu failover procedures. Nếu standby fail thì không cần failover; chỉ cần phục hồi hoặc tạo lại standby.  
Reference: [Failover](https://www.postgresql.org/docs/current/warm-standby-failover.html)

## Hiểu rất dễ

- **Primary chết** → phải chọn một standby lên thay.
- **Standby chết** → primary vẫn chạy, chỉ mất bớt khả năng dự phòng.

## Ví dụ

Ban đầu:

- `postgres-0` = primary
- `postgres-1` = standby
- `postgres-2` = standby

Nếu `postgres-0` chết:

- `postgres-1` có thể được promote thành primary mới.

Nếu `postgres-2` chết:

- không cần đổi primary; chỉ cần tạo lại standby sau.

---

# 10) Script trong ConfigMap đang làm gì?

Trong bài của DevOpsCube, script làm mấy việc sau:

1. kiểm tra pod sắp dừng là primary hay follower,
2. nếu là primary thì trì hoãn việc pod rời cluster cho đến khi follower cũ được promote thành primary mới,
3. mục tiêu là luôn duy trì ít nhất một master/primary có khả năng write.

Reference: [How to Deploy PostgreSQL Statefulset Cluster on Kubernetes](https://devopscube.com/deploy-postgresql-statefulset/)

Trong script còn có đoạn:

- `postgresql_stop` để dừng PostgreSQL graceful,
- rồi nếu node hiện tại là primary thì `retry_while is_new_primary_ready 10 5`.

**Câu chốt:**  
Script đó không chờ pod mới lên.  
Script đó **chờ cluster DB phát hiện có primary mới**.

---

# 11) “Chờ” bằng cách nào? Nó có gọi API bảo Kubernetes đợi không?

**Không.**

Kubernetes đã định nghĩa sẵn `PreStop` là:

- được gọi ngay trước khi container bị terminate,
- hook phải chạy xong trước khi `TERM` được gửi,
- nếu hook treo, pod nằm ở trạng thái `Terminating` cho tới khi hết `terminationGracePeriodSeconds`.

Reference: [Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)

## Nghĩa là gì?

Script không cần gọi:

> “Ê Kubernetes, đợi tao chút.”

Thay vào đó:

- Kubernetes bắt đầu stop pod,
- thấy có `preStop`,
- nên **tự chờ script chạy xong**,
- script càng chưa exit thì quá trình terminate càng chưa đi tiếp,
- nhưng vẫn bị giới hạn bởi `terminationGracePeriodSeconds`.

## Ví dụ

Script lặp kiểm tra 10 lần, mỗi lần nghỉ 5 giây:

- chưa thấy primary mới → tiếp tục lặp,
- script chưa xong → kubelet chưa gửi `TERM`,
- pod bị giữ ở pha terminating một lúc.

---

# 12) ConfigMap có vai trò gì ở đây?

ConfigMap ở đây chỉ là nơi chứa file script để mount vào pod.  
Nó không phải “bộ não HA”, cũng không phải “service phát hiện lỗi”.

## Hiểu đơn giản

- Script được viết trong ConfigMap.
- Pod mount ConfigMap thành file `/pre-stop.sh`.
- Khi pod sắp stop, kubelet chạy file đó bằng `preStop`.

Reference: [How to Deploy PostgreSQL Statefulset Cluster on Kubernetes](https://devopscube.com/deploy-postgresql-statefulset/)

**Câu chốt:**  
ConfigMap chỉ là **cách nhét script vào container** mà không cần build lại image.

---

# 13) “Chờ” đó hiệu quả như thế nào?

Bitnami chart `postgresql-ha` mô tả tham số `postgresql.preStopDelayAfterPgStopSeconds` như sau:

- đây là số giây tối thiểu `preStop` chờ sau khi PostgreSQL instance đã dừng,
- dùng để delay PostgreSQL pod termination,
- cho Pgpool-II thời gian phát hiện node đi xuống,
- để node PostgreSQL được đăng ký đúng trong Pgpool-II, nhất là primary flag,
- và phải nhỏ hơn `terminationGracePeriodSeconds`.

Reference: [bitnami/postgresql-ha values.yaml](https://github.com/bitnami/charts/blob/main/bitnami/postgresql-ha/values.yaml)

## Ý nghĩa thực tế

Cái chờ này tạo ra **khoảng đệm bàn giao**:

- primary cũ chưa biến mất quá nhanh,
- proxy/failover manager có thêm thời gian cập nhật “primary mới là ai”,
- giảm nguy cơ app vẫn cố đâm write vào node vừa chết.

**Nhưng:** nó không đảm bảo zero-error tuyệt đối.  
Nó chỉ làm graceful hơn trong các lần shutdown có kiểm soát.

---

# 14) Khi nào `preStop` hữu ích nhất?

Kubernetes docs nói `PreStop` được gọi khi container bị terminate do:

- API request,
- management event,
- probe failure,
- preemption,
- resource contention, v.v.

và hook phải xong trước khi gửi `TERM`.  
Reference: [Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)

## Hữu ích nhất trong các trường hợp có kiểm soát

- `kubectl drain`
- rolling update
- delete pod có kiểm soát
- restart có kiểm soát

Kubernetes docs về drain nói `kubectl drain` dùng để **safely evict all pods** trước khi bảo trì node, cho phép pod graceful terminate và tôn trọng PodDisruptionBudget.  
Reference: [Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)

## Ví dụ

Node cần bảo trì:

1. chạy `kubectl drain`,
2. pod DB bị evict một cách graceful,
3. `preStop` được gọi,
4. script chờ hệ thống HA kịp đổi primary,
5. node mới được hạ xuống.

---

# 15) Khi nào `preStop` không cứu được?

Nếu là hard failure:

- node mất điện,
- kernel panic,
- node treo cứng,
- network partition xấu,
- process chết đột ngột trước flow terminate,

thì có thể **không có cơ hội chạy `preStop` đúng nghĩa**.

Kubernetes chỉ hứa `PreStop` trong flow terminate bình thường; grace period cũng bắt đầu trước khi hook chạy.  
Reference: [Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)

PostgreSQL docs cũng nói PostgreSQL **không tự cung cấp system software để phát hiện primary failure và notify standby**; cần tool bên ngoài hỗ trợ failover.  
Reference: [Failover](https://www.postgresql.org/docs/current/warm-standby-failover.html)

**Câu chốt:**  
`preStop` là công cụ cho **graceful termination**, không phải thuốc chữa cho mọi kiểu chết đột ngột.

---

# 16) Khi primary chết thì app có lỗi không?

**Có thể có.**

PostgreSQL docs nói khi primary fail thì standby mới bắt đầu failover procedures; sau failover chỉ còn một server hoạt động, và muốn quay lại trạng thái bình thường thì phải tái tạo standby.  
Reference: [Failover](https://www.postgresql.org/docs/current/warm-standby-failover.html)

## Điều app có thể thấy

- connection reset
- timeout
- write failed
- một khoảng ngắn không có write target hợp lệ

## Vì sao?

Vì DB HA không giống web app LB.  
Cần thời gian để:

- phát hiện primary chết,
- promote standby,
- cập nhật routing/proxy,
- app reconnect/retry.

**Câu chốt:**  
HA database thường là **giảm downtime**, chứ không phải hứa **không bao giờ có lỗi đúng khoảnh khắc failover**.

---

# 17) “Nếu primary bị nghẽn CPU/RAM thì standby có chắc cứu được không?”

**Không chắc.**

Replication/failover giúp nhiều nhất khi:

- primary chết hẳn,
- node chứa primary down,
- primary cần maintenance,
- cần read scaling bằng hot standby.

Reference: [Failover](https://www.postgresql.org/docs/current/warm-standby-failover.html)

Nó không tự chữa được các bệnh như:

- cả cluster đều thiếu CPU/RAM,
- query quá tệ,
- storage chậm,
- lock contention,
- replication lag lớn.

PostgreSQL docs chỉ ra bài toán căn bản là đồng bộ write giữa các server là khó; không có một giải pháp duy nhất xóa bỏ hoàn toàn tác động của đồng bộ cho mọi workload.  
Reference: [High Availability, Load Balancing, and Replication](https://www.postgresql.org/docs/current/high-availability.html)

**Câu chốt:**  
HA chủ yếu tăng **availability** và có thể tăng **read scalability**; nó không tự chữa mọi vấn đề hiệu năng.

---

# 18) “StatefulSet thay thế” thực chất là gì?

Rất nhiều người nhầm chữ “thay thế”.

## StatefulSet thay thế ở tầng pod

Nếu `postgres-0` chết, StatefulSet cố tạo lại **pod `postgres-0`** với identity cũ.  
Reference: [StatefulSets | Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)

## PostgreSQL failover thay thế ở tầng vai trò

Nếu primary cũ chết, một standby khác được promote thành primary mới.  
Reference: [Failover](https://www.postgresql.org/docs/current/warm-standby-failover.html)

Đây là **hai loại thay thế khác nhau**:

- thay pod
- thay vai trò DB

**Script `preStop` đang chờ cái thứ hai**, không phải cái thứ nhất.  
Reference: [How to Deploy PostgreSQL Statefulset Cluster on Kubernetes](https://devopscube.com/deploy-postgresql-statefulset/)

---

# 19) Ví dụ timeline hoàn chỉnh

Giả sử có:

- `postgres-0` = primary
- `postgres-1` = standby
- `postgres-2` = standby

Và bạn chạy `kubectl drain node-A`, nơi đang chứa `postgres-0`.

## Timeline

**T0**: drain bắt đầu, pod bị evict graceful.  
Reference: [Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)

**T1**: kubelet gọi `preStop`.  
Reference: [Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)

**T2**: script kiểm tra thấy “ta đang là primary”.  
Reference: [How to Deploy PostgreSQL Statefulset Cluster on Kubernetes](https://devopscube.com/deploy-postgresql-statefulset/)

**T3**: script gọi `postgresql_stop` rồi lặp chờ `is_new_primary_ready`.  
Reference: [How to Deploy PostgreSQL Statefulset Cluster on Kubernetes](https://devopscube.com/deploy-postgresql-statefulset/)

**T4**: repmgr/Pgpool-II phát hiện node cũ down và promote `postgres-1` thành primary mới.  
Reference: [bitnami/postgresql-ha values.yaml](https://github.com/bitnami/charts/blob/main/bitnami/postgresql-ha/values.yaml)

**T5**: script thấy primary mới đã tồn tại → exit.  
Reference: [How to Deploy PostgreSQL Statefulset Cluster on Kubernetes](https://devopscube.com/deploy-postgresql-statefulset/)

**T6**: kubelet mới gửi `TERM` nếu container còn cần stop tiếp; pod cũ kết thúc.  
Reference: [Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)

**T7**: StatefulSet nếu cần sẽ tạo lại pod đúng identity như `postgres-0` để quay lại làm standby sau.  
Reference: [StatefulSets | Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)

---

# 20) Những câu hỏi “ngơ ngơ” nhưng rất đúng

## “StatefulSet có tự replicate DB không?”

Không. StatefulSet chỉ cho pod identity/storage/order ổn định. Replication là việc của PostgreSQL + tool HA.  
Reference: [StatefulSets | Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)

## “Service có tự chọn pod DB nhàn nhất không?”

Không phải theo nghĩa hiểu CPU/RAM rồi chọn node tốt nhất; DB HA còn bị ràng buộc vai trò primary/standby.  
Reference: [High Availability, Load Balancing, and Replication](https://www.postgresql.org/docs/current/high-availability.html)

## “Script có gọi API Kubernetes để xin chờ không?”

Không. Kubernetes tự chờ vì `preStop` là bước blocking trước `TERM`.  
Reference: [Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)

## “Script chờ pod mới lên à?”

Không. Nó chờ **primary mới** được nhìn thấy.  
Reference: [How to Deploy PostgreSQL Statefulset Cluster on Kubernetes](https://devopscube.com/deploy-postgresql-statefulset/)

## “Nếu node chết cứng thì `preStop` có cứu được không?”

Không chắc; `preStop` mạnh nhất trong shutdown có kiểm soát.  
Reference: [Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)

## “Failover xong app có chắc không lỗi?”

Không chắc. Có thể vẫn có timeout/lỗi ngắn trong lúc failover.  
Reference: [Failover](https://www.postgresql.org/docs/current/warm-standby-failover.html)

---

# 21) Bản học siêu ngắn để nhớ lòng

1. `Deployment + PVC` = giữ data persistent cho DB đơn giản/single-instance.
2. `StatefulSet` = giữ pod identity + storage ổn định cho DB nhiều node.
3. `StatefulSet` không tự replicate database.
4. `Primary` = node DB được ghi.
5. `Standby` = node theo sau primary.
6. `Hot standby` có thể nhận read-only queries.
7. `Failover` = primary chết thì standby lên thay.
8. `ConfigMap` trong bài chỉ để mang script vào pod.
9. `preStop` chạy trước `TERM`; kubelet tự chờ hook xong.
10. Script trong bài chờ **primary mới**, không chờ pod mới.
11. Cái chờ đó hữu ích nhất khi drain / rolling update / graceful termination.
12. Nó không chữa được hard crash hay mọi lỗi network/split-brain.
13. HA DB giúp availability, nhưng không tự chữa mọi vấn đề hiệu năng.

---

# 22) Một ví dụ cực đời thường để khóa kiến thức

Hãy tưởng tượng một cửa hàng có:

- 1 quản lý chính = primary
- 2 phó quản lý = standby

## StatefulSet là gì?

Là bộ phận nhân sự đảm bảo:

- mỗi người có mã nhân viên cố định,
- hồ sơ làm việc không bị lẫn,
- khi người A nghỉ thì tuyển lại đúng vị trí A.

## PostgreSQL HA là gì?

Là quy định vận hành:

- chỉ quản lý chính được ký giấy xuất kho,
- phó quản lý chỉ theo dõi và hỗ trợ,
- nếu quản lý chính nghỉ thì phải chỉ định một phó lên thay.

## `preStop` script là gì?

Là quy trình bàn giao:

- nếu người sắp nghỉ là phó → đi luôn.
- nếu người sắp nghỉ là quản lý chính → đợi đến khi có quản lý mới rồi mới cho ra khỏi ca.

Đó chính là ý nghĩa của chữ **“chờ”** và **“thay thế”** trong toàn bộ bài toán này.

---

# 23) References

1. [Deployments | Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
2. [StatefulSets | Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
3. [Persistent Volumes | Kubernetes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
4. [Run a Single-Instance Stateful Application](https://kubernetes.io/docs/tasks/run-application/run-single-instance-stateful-application)
5. [Run a Replicated Stateful Application](https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application)
6. [High Availability, Load Balancing, and Replication](https://www.postgresql.org/docs/current/high-availability.html)
7. [Hot Standby](https://www.postgresql.org/docs/current/hot-standby.html)
8. [Failover](https://www.postgresql.org/docs/current/warm-standby-failover.html)
9. [How to Deploy PostgreSQL Statefulset Cluster on Kubernetes](https://devopscube.com/deploy-postgresql-statefulset/)
10. [Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)
11. [bitnami/postgresql-ha values.yaml](https://github.com/bitnami/charts/blob/main/bitnami/postgresql-ha/values.yaml)
12. [Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)
13. [WordPress and MySQL with Persistent Volumes](https://kubernetes.io/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/)
