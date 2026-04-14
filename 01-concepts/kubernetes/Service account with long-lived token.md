### 1. Khái niệm và Mục đích

**Service Account (SA)** là một loại tài khoản dành cho các đối tượng không phải con người (như Pods hoặc ứng dụng bên ngoài) để xác thực với API Server của Kubernetes.

**Tại sao cần dùng Service Account cho API?**

- **Truy cập từ bên thứ ba:** Các công cụ như Prometheus cần quyền đọc dữ liệu từ cụm (metrics, thông tin Pod) để giám sát.
- **Nguyên tắc Quyền hạn tối thiểu (PoLP):** Thay vì dùng tài khoản quản trị (admin), bạn chỉ cấp đúng các quyền cần thiết cho từng ứng dụng cụ thể.
- **Token dài hạn (Long-lived Token):** Giúp ứng dụng duy trì kết nối ổn định mà không cần cấp lại thông tin xác thực thường xuyên.

---

### 2. Quy trình thiết lập từng bước

#### Bước 1: Tạo Namespace và Service Account

Nên tạo một namespace riêng để quản lý các công cụ vận hành.

```
# Tạo namespace
kubectl create namespace devops-tools

# Tạo Service Account
kubectl create serviceaccount api-service-account -n devops-tools
```

#### Bước 2: Tạo ClusterRole (Danh sách quyền hạn toàn cụm)

Nếu ứng dụng cần truy cập tài nguyên trên toàn bộ cụm (không chỉ trong 1 namespace), bạn cần dùng **ClusterRole**. Ví dụ cấu hình cho phép xem (get, list, watch) hầu hết các tài nguyên quan trọng:

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: api-cluster-role
rules:
- apiGroups: ["", "apps", "autoscaling", "batch"]
  resources: ["pods", "nodes", "deployments", "services", "namespaces"]
  verbs: ["get", "list", "watch"] # Chỉ cấp quyền đọc để an toàn,
```

#### Bước 3: Liên kết bằng ClusterRoleBinding

Kết nối Service Account vừa tạo với danh sách quyền hạn trong ClusterRole,.

```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: api-cluster-role-binding
subjects:
- kind: ServiceAccount
  name: api-service-account
  namespace: devops-tools
roleRef:
  kind: ClusterRole
  name: api-cluster-role
  apiGroup: rbac.authorization.k8s.io
EOF
```

#### Bước 4: Tạo Long-lived Token (Mã xác thực dài hạn)

Để gọi API qua HTTP, bạn cần một Token được lưu trữ trong một **Secret**.

1. **Tạo Secret:**
    
    ```
    apiVersion: v1
    kind: Secret
    metadata:
      name: api-service-account-token
      namespace: devops-tools
      annotations:
        kubernetes.io/service-account.name: api-service-account # Liên kết với SA
    type: kubernetes.io/service-account-token
    ```
    
2. **Lấy Token đã giải mã:**
    
    ````
    kubectl get secret api-service-account-token -o=jsonpath='{.data.token}' -n devops-tools | base64 --decode
    ```,.
    
    ````
    

---

### 3. Kiểm tra và Sử dụng

**Kiểm tra quyền bằng kubectl:** Bạn có thể xác nhận quyền hạn mà không cần chạy ứng dụng thực tế bằng lệnh `can-i`:

- `kubectl auth can-i get pods --as=system:serviceaccount:devops-tools:api-service-account`.
- Nếu trả về **yes**, nghĩa là thiết lập đã thành công.

**Sử dụng qua API Call (CURL):** Sau khi có Token và địa chỉ IP của cụm (Endpoint), bạn có thể truy cập từ bất cứ đâu:

````
curl -k https://<CLUSTER_IP>/api/v1/namespaces -H "Authorization: Bearer <TOKEN_CỦA_BẠN>"
```,.

---

### 4. Lưu ý quan trọng cho người mới (Best Practices)
*   **Không dùng Service Account mặc định:** Tuyệt đối không gắn ClusterRole vào tài khoản `default` vì mọi Pod trong namespace đó sẽ mặc nhiên có quyền cao, gây rủi ro bảo mật.
*   **Sử dụng công cụ quản lý Secret:** Nên dùng các công cụ như **Hashicorp Vault** để lưu trữ và chia sẻ các Token này thay vì để lộ trong các file văn bản thuần túy.
*   **Hạn chế quyền hạn:** Luôn bắt đầu với quyền nhỏ nhất (chỉ đọc) và chỉ mở rộng khi thực sự cần thiết,.

Hy vọng bản tổng hợp chi tiết này giúp bạn nắm vững cách quản lý truy cập API trong Kubernetes! Bạn có muốn tôi hướng dẫn thêm về cách tích hợp Token này vào một ứng dụng cụ thể không?
````

### ==Giải thích cho người mới: Service Account vs. Human Account==

Để dễ hình dung, bạn hãy tưởng tượng cụm Kubernetes giống như một tòa nhà văn phòng cao cấp:

- **Human Account (Tài khoản người dùng):** Giống như nhân viên có thẻ tên, dùng để đăng nhập vào UI (như Rancher, Dashboard) để làm việc trực tiếp.
- **Service Account (SA - Tài khoản dịch vụ):** Giống như một **"chiếc thẻ từ"** dành riêng cho các "robot" hoặc "ứng dụng".
    - **Mục đích:** Giúp các ứng dụng bên ngoài (như Prometheus, ứng dụng web của bạn) có thể "nói chuyện" với hệ thống Kubernetes mà không cần sự can thiệp của con người.
    - **Lợi ích:** Bạn không cần phải `exec` (chui) vào bất kỳ Pod nào bên trong. Bạn có thể đứng từ máy tính cá nhân ở nhà, dùng "chiếc thẻ từ" này để điều khiển cụm K8s từ xa qua internet.