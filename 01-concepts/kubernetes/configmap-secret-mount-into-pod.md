---
doc_id: configmap-secret-mount-into-pod
title: ConfigMap, Secret và cơ chế mount vào Pod
legacy_title: ConfigMap Secret và cơ chế mount vào Pod
type: concept-note
domain: kubernetes
keywords:
  - configmap
  - secret
  - mount
  - env
  - volume
  - pod
status: active
---

# ConfigMap, Secret và cơ chế mount vào Pod

Nguồn tham khảo:
- [How to Deploy PostgreSQL Statefulset in Kubernetes With High Availability](https://devopscube.com/deploy-postgresql-statefulset/)
- [Kubernetes Docs - Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- [Kubernetes Docs - Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)
- [Kubernetes Docs - Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)

## Hiểu đúng bản chất

==`ConfigMap` và `Secret` không phải chỉ là nơi lưu biến môi trường.==

Bản chất của chúng là:
- ==`ConfigMap` dùng để chứa dữ liệu cấu hình không nhạy cảm.==
- ==`Secret` dùng để chứa dữ liệu cấu hình nhạy cảm như password, token, private key.==
- ==Cả hai đều là object dữ liệu trong Kubernetes, lưu theo dạng key-value.==

Điều quan trọng là:

<span style="color:red">`ConfigMap` và `Secret` không tự động biến thành biến môi trường.</span>

Chúng chỉ là **nguồn dữ liệu cấu hình**. Muốn Pod dùng được, bạn phải khai báo cách đưa dữ liệu đó vào container.

## Có những cách nào để đưa ConfigMap hoặc Secret vào container?

Thông thường có 3 cách:

### 1. Đưa vào dưới dạng biến môi trường

Ví dụ:

```yaml
env:
  - name: DB_HOST
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: db_host
```

Hoặc với Secret:

```yaml
env:
  - name: POSTGRES_PASSWORD
    valueFrom:
      secretKeyRef:
        name: postgres-secrets
        key: postgresql-password
```

Khi container khởi động, Kubernetes sẽ inject giá trị đó thành env var cho process trong container.

==Đây là cách phổ biến nhất, nhưng không phải cách duy nhất.==

### 2. Mount thành file trong container

Ví dụ với ConfigMap:

```yaml
volumes:
  - name: app-config
    configMap:
      name: my-config
```

và:

```yaml
volumeMounts:
  - name: app-config
    mountPath: /etc/config
```

Lúc này, key trong `ConfigMap` sẽ xuất hiện thành file trong thư mục `/etc/config`.

Ví dụ:

```yaml
data:
  app.properties: |-
    server.port=8080
    app.mode=dev
```

thì trong container sẽ có file:

```text
/etc/config/app.properties
```

==Đây là lý do vì sao shell script, file `.conf`, `.yaml`, `.properties` hoàn toàn có thể nằm trong `ConfigMap`.==

### 3. Ứng dụng tự đọc qua Kubernetes API

Cách này ít phổ biến hơn trong ứng dụng đơn giản, nhưng vẫn có thể xảy ra. Ứng dụng hoặc controller có thể gọi Kubernetes API để lấy dữ liệu từ `ConfigMap`/`Secret`.

## Khi nào ConfigMap hoặc Secret được đưa vào Pod?

Nếu bạn khai báo `env`, `envFrom`, `volumes`, `volumeMounts`, thì việc đưa dữ liệu vào Pod xảy ra trong quá trình container được khởi động.

Theo Kubernetes docs về `Volumes`:

==Khi Pod được launch, process trong container sẽ nhìn thấy filesystem gồm nội dung image cộng với các volume đã được mount vào.==

Điều đó nghĩa là:

1. Pod được tạo.
2. Scheduler gán Pod vào node.
3. Kubelet chuẩn bị container.
4. Nếu có `ConfigMap volume` hoặc `Secret volume`, kubelet mount chúng vào.
5. Container start.
6. Ứng dụng bên trong container nhìn thấy file đã có sẵn tại path được mount.

<span style="color:red">Mount file không có nghĩa là file sẽ tự chạy.</span>

Mount chỉ có nghĩa là file đó xuất hiện trong filesystem của container.

Chính xác hơn:

- kubelet chịu trách nhiệm chuẩn bị pod và mount `ConfigMap`/`Secret` vào container
- nhưng kubelet không tự suy luận rằng "thấy có file `.sh` thì phải chạy file đó"
- script chỉ chạy khi `podSpec` hoặc process trong container có chỉ định rõ cách dùng file đó

Ví dụ các trường hợp script thật sự được chạy:

- container `command` / `args` gọi tới file script đã mount
- ứng dụng bên trong container tự đọc và chạy file đó
- lifecycle hook như `preStop` hoặc `postStart` được cấu hình để exec script
- `exec probe` được cấu hình để chạy lệnh trong container

## Vậy script trong ConfigMap có hợp lý không?

==Có, hoàn toàn hợp lý.==

Vì shell script cũng chỉ là text file. Mà `ConfigMap` rất phù hợp để chứa text file như:
- file cấu hình `.conf`
- file `.yaml`
- file `.properties`
- script `.sh`
- file template

Ví dụ:

```yaml
data:
  pre-stop.sh: |-
    #!/bin/bash
    echo "container is stopping"
```

Khi mount vào Pod, key `pre-stop.sh` trở thành một file thật trong container.

## Tại sao không phải CronJob hay webhook?

Đây là điểm rất dễ nhầm.

### `ConfigMap` không quyết định *khi nào chạy*

`ConfigMap` chỉ quyết định:
- lưu dữ liệu gì
- đưa dữ liệu đó vào container bằng cách nào

Nó **không phải** cơ chế trigger.

### `CronJob` dùng để làm gì?

`CronJob` dùng cho tác vụ chạy **theo lịch**.

Ví dụ:
- 2 giờ sáng backup database
- mỗi 30 phút dọn log
- mỗi ngày gửi báo cáo

`CronJob` không phải cơ chế "pod sắp chết thì chạy script".

### `Webhook` dùng để làm gì?

Webhook thường dùng cho:
- nhận HTTP callback
- admission webhook trong Kubernetes
- trigger từ hệ thống bên ngoài

Nó cũng không phải chỗ phù hợp để gắn logic shutdown của một Pod cụ thể.

### Trong bài PostgreSQL kia, thứ quyết định thời điểm chạy là gì?

==Chính là `lifecycle.preStop`.==

## Chuỗi hoạt động đầy đủ trong bài PostgreSQL StatefulSet

### 1. ConfigMap chứa script

Trong bài:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-configmap
data:
  pre-stop.sh: |-
    #!/bin/bash
    ...
```

Ý nghĩa:
- `ConfigMap` tên `postgres-configmap`
- có một key tên `pre-stop.sh`
- value của key đó là nội dung script shell

==Ở bước này, script mới chỉ đang được lưu trong Kubernetes, chưa chạy.==

### 2. StatefulSet mount script vào container

Trong bài:

```yaml
volumeMounts:
  - name: hooks-scripts
    mountPath: /pre-stop.sh
    subPath: pre-stop.sh
volumes:
  - name: hooks-scripts
    configMap:
      name: postgres-configmap
      defaultMode: 0755
```

Giải thích:
- lấy dữ liệu từ `ConfigMap` tên `postgres-configmap`
- lấy đúng key `pre-stop.sh`
- mount nó thành file `/pre-stop.sh` trong container
- `defaultMode: 0755` để file có quyền thực thi

==Ở bước này, script đã xuất hiện trong container dưới dạng file.==

<span style="color:red">Nhưng nó vẫn chưa tự chạy.</span>

### 3. `preStop` hook gọi script lúc container sắp bị terminate

Trong bài:

```yaml
lifecycle:
  preStop:
    exec:
      command:
        - /pre-stop.sh
```

Đây là phần quan trọng nhất.

Theo tài liệu Kubernetes:
- ==`PreStop` hook được gọi ngay trước khi container bị terminate.==
- ==Hook này phải chạy xong trước khi tín hiệu dừng chính thức được gửi vào container.==

Nghĩa là:

1. Pod bị yêu cầu dừng.
2. Kubelet bắt đầu quá trình terminate.
3. Vì container có `preStop`, kubelet yêu cầu runtime exec `/pre-stop.sh` bên trong container.
4. Script chạy xong.
5. Sau đó Kubernetes mới tiếp tục quá trình dừng container.

<span style="color:red">Đây chính là chỗ mang nghĩa: "ok pod tao sắp chết rồi, chạy script đó đi".</span>

Điểm cần phân biệt:

- kubelet không tự chạy "mọi script nằm trong ConfigMap/Secret"
- kubelet chỉ kích hoạt script khi Pod đã khai báo rõ hook hoặc lệnh cần chạy

## Khi nào `preStop` được gọi?

`preStop` không chạy khi Pod vừa start.

Nó chạy khi container **sắp bị terminate**, ví dụ:
- `kubectl delete pod`
- rolling update
- scale down
- Pod bị thay thế
- một số tình huống quản lý vòng đời khiến container phải dừng

Theo Kubernetes docs:
- `preStop` chạy trước khi container bị kết thúc
- grace period bắt đầu tính trước khi hook này chạy

Nghĩa là script shutdown phải đủ nhanh và hợp lý.

## Trong bài PostgreSQL, script đó chạy để làm gì?

Script `pre-stop.sh` dùng các thư viện sẵn có của Bitnami:

```bash
. /opt/bitnami/scripts/liblog.sh
. /opt/bitnami/scripts/libpostgresql.sh
. /opt/bitnami/scripts/librepmgr.sh
```

Mục tiêu của script:
- xác định Pod hiện tại là `primary` hay `standby`
- dừng PostgreSQL một cách graceful
- nếu Pod đang là `primary`, chờ cho đến khi có `primary` mới được promote

==Mục đích là tránh việc primary chết quá đột ngột làm cluster mất khả năng ghi ngay lập tức.==

## ConfigMap và Secret khác nhau thế nào?

### ConfigMap

Dùng cho dữ liệu không nhạy cảm, ví dụ:
- hostname
- port
- file cấu hình
- script vận hành nhỏ
- feature flags

Ví dụ:

```yaml
data:
  APP_MODE: production
  nginx.conf: |-
    server {
      listen 80;
    }
```

### Secret

Dùng cho dữ liệu nhạy cảm, ví dụ:
- password
- token
- certificate
- private key

Ví dụ:

```yaml
env:
  - name: POSTGRES_PASSWORD
    valueFrom:
      secretKeyRef:
        name: postgres-secrets
        key: postgresql-password
```

==Hiểu ngắn gọn:==
- ==`ConfigMap` = cấu hình không nhạy cảm==
- ==`Secret` = cấu hình nhạy cảm==

## Điều dễ nhầm nhất

<span style="color:red">Sai lầm phổ biến là nghĩ `ConfigMap` chỉ dùng cho env vars.</span>

Hiểu đúng phải là:

- `ConfigMap` là nơi chứa dữ liệu cấu hình không nhạy cảm
- `Secret` là nơi chứa dữ liệu cấu hình nhạy cảm
- cả hai có thể đi vào container bằng env vars hoặc file
- file được mount vào container không tự chạy
- muốn script chạy thì cần có cơ chế gọi nó, ví dụ `command`, `entrypoint`, `lifecycle hook`, hoặc ứng dụng tự gọi

## Cách nhớ nhanh

### Nếu mục tiêu là đưa giá trị đơn giản vào app

Ví dụ:
- `DB_HOST`
- `APP_MODE`
- `LOG_LEVEL`

=> thường dùng `env` hoặc `envFrom`

### Nếu mục tiêu là đưa file cấu hình hoặc script vào app

Ví dụ:
- `nginx.conf`
- `app.yaml`
- `postgresql.conf`
- `pre-stop.sh`

=> thường dùng `ConfigMap volume` hoặc `Secret volume`

### Nếu mục tiêu là chạy script khi Pod sắp dừng

=> dùng:
- `ConfigMap` để chứa script
- `volumeMount` để mount script vào container
- `lifecycle.preStop` để gọi script đúng thời điểm

## Sơ đồ tư duy ngắn gọn

```text
ConfigMap/Secret
        |
        |  (env hoặc volume)
        v
Pod / Container
        |
        |-- env var -> ứng dụng đọc như biến môi trường
        |
        |-- mounted file -> ứng dụng hoặc hook đọc/chạy file
                         |
                         |-- nếu có lifecycle.preStop
                         v
                    script chạy khi container sắp terminate
```

## Kết luận

==`ConfigMap` và `Secret` là nguồn dữ liệu cấu hình cho Pod, không phải chỉ là nơi chứa env vars.==

==Chúng có thể được đưa vào container dưới dạng biến môi trường hoặc file được mount.==

==Trong bài PostgreSQL StatefulSet, script nằm trong `ConfigMap`, được mount vào container thành file `/pre-stop.sh`, và chỉ chạy khi `lifecycle.preStop` gọi nó lúc Pod sắp bị terminate.==
