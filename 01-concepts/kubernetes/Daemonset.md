### 1. DaemonSet là gì? (Khái niệm cơ bản)

Trong Kubernetes, **DaemonSet** là một đối tượng đảm bảo rằng **một bản sao của Pod sẽ chạy trên mỗi nút (worker node)** trong cụm (cluster).

- **Đặc điểm chính:** Khi bạn thêm một nút mới vào cụm, DaemonSet sẽ tự động tạo một Pod trên nút đó. Ngược lại, khi nút bị xóa, Pod cũng sẽ bị xóa theo.
- **Sự khác biệt:** Khác với Deployment (nơi bạn quyết định số lượng bản sao), số lượng Pod của DaemonSet mang tính chất động và phụ thuộc trực tiếp vào số lượng nút trong cụm.
- **Mục đích:** Thường dùng để chạy các dịch vụ hệ thống (daemons) cần có mặt trên mọi máy chủ để hỗ trợ hạ tầng.

---

### 2. Các trường hợp sử dụng thực tế (Use Cases)

DaemonSet thường được dùng cho các tác vụ mang tính chất "nền tảng":

- **Thu thập nhật ký (Logging):** Chạy các agent như `fluentd`, `logstash` hoặc `fluentbit` trên mỗi nút để gom log về trung tâm.
- **Giám sát (Monitoring):** Triển khai các agent như `Prometheus Node Exporter` để thu thập dữ liệu hiệu năng của từng node.
- **Bảo mật:** Chạy các công cụ quét lỗ hổng hoặc hệ thống phát hiện xâm nhập trên các node cụ thể hoặc toàn bộ cụm.
- **Quản lý mạng & Lưu trữ:** Các plugin mạng như `Calico CNI` hoặc các plugin lưu trữ cần chạy trên mọi node để cung cấp dịch vụ cho toàn cụm.

---

### 3. Ví dụ chi tiết về File cấu hình (YAML)

Dưới đây là ví dụ về một file `daemonset.yaml` dùng để triển khai trình thu thập log **Fluentd**:

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging # Chạy trong namespace riêng để quản lý
  labels:
    app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources: # Giới hạn tài nguyên để không ảnh hưởng node
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

**Giải thích các thành phần chính:**

1. **apiVersion & kind:** Xác định đây là đối tượng DaemonSet phiên bản `apps/v1`.
2. **metadata:** Tên và nhãn (label) để định danh DaemonSet.
3. **spec.selector:** Giúp DaemonSet biết nó quản lý các Pod nào (dựa trên label).
4. **spec.template:** Định nghĩa cấu hình của Pod sẽ được tạo ra. Nó bao gồm hình ảnh (image), tài nguyên (CPU/RAM) và các ổ đĩa (volumes).
5. **hostPath:** Cho phép Pod truy cập trực tiếp vào thư mục log trên máy chủ vật lý/máy ảo (node).

---

### 4. Cách kiểm soát vị trí chạy Pod

Mặc dù mặc định DaemonSet chạy trên tất cả worker node, bạn có thể giới hạn nó chỉ chạy trên một số node nhất định:

- **nodeSelector:** Chỉ định Pod chỉ chạy trên các node có nhãn (label) cụ thể. Ví dụ: Chỉ chạy trên các node có GPU.
- **Node Affinity:** Một cách chọn node linh hoạt và mạnh mẽ hơn `nodeSelector`. Bạn có thể yêu cầu (required) hoặc ưu tiên (preferred) các node nhất định.
- **Taints và Tolerations:**
    - **Taint:** Dùng để "đánh dấu" một node là không phù hợp cho các Pod thông thường (ví dụ: node đang bảo trì).
    - **Toleration:** Nếu bạn muốn DaemonSet chạy trên một node đã bị Taint (như Control Plane), bạn phải thêm Toleration vào cấu hình Pod của DaemonSet.

---

### 5. Quản lý và Cập nhật

- **Chiến lược cập nhật (Update Strategy):**
    - **RollingUpdate (Mặc định):** Khi bạn thay đổi cấu hình, K8s sẽ lần lượt xóa Pod cũ và tạo Pod mới trên từng node để đảm bảo dịch vụ không bị gián đoạn.
    - **OnDelete:** Chỉ tạo Pod mới khi bạn xóa Pod cũ một cách thủ công.
- **Lệnh cơ bản:**
    - Triển khai: `kubectl apply -f daemonset.yaml`
    - Xem trạng thái: `kubectl get ds -n <namespace>`
    - Hoàn tác (Rollback): `kubectl rollout undo daemonset <tên-ds>`

---

### 6. Các lưu ý quan trọng cho người mới (Best Practices)

- **Resource Limits:** Luôn thiết lập giới hạn CPU và RAM ở mức tối thiểu cần thiết vì các Pod này chạy trên **mọi**nút, nếu chiếm quá nhiều tài nguyên sẽ ảnh hưởng đến ứng dụng chính.
- **Restart Policy:** Phải luôn được thiết lập là `Always` (hoặc để trống vì mặc định là Always).
- **PriorityClass:** Với các dịch vụ hệ thống quan trọng, nên thiết lập mức ưu tiên cao (PriorityClass) để tránh việc Pod bị K8s xóa khi node bị thiếu tài nguyên.
- **Isolation:** Nên tách các DaemonSet vào các `namespace` riêng biệt để dễ quản lý và bảo mật.

