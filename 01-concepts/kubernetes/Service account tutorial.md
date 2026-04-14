### 1. Khái niệm cơ bản (Bạn cần biết gì?)

Bình thường, các Pod chạy trong K8s không có quyền tự ý thay đổi hay xem thông tin của các tài nguyên khác. Tuy nhiên, đôi khi ứng dụng của bạn cần quyền để "quản lý" cụm (ví dụ: một ứng dụng tự động kiểm tra trạng thái các Pod khác). Để làm việc này an toàn, K8s sử dụng 3 thành phần:

- **Service Account:** Là "danh tính" (identity) dành cho Pod. Thay vì con người đăng nhập, Pod sẽ sử dụng Service Account để "đăng nhập" vào hệ thống.
- **Role (Vai trò):** Là bản danh sách các quyền hạn. Nó định nghĩa cụ thể: "Được phép làm gì" (xem, xóa, tạo) trên "tài nguyên nào" (Pod, ConfigMap, Deployment).
- **RoleBinding:** Là "sợi dây liên kết". Nó gán một **Role** cụ thể cho một **Service Account** nhất định.

**Lưu ý:** `Role` chỉ có tác dụng trong phạm vi một **Namespace** (không gian tên) nhất định. Nếu muốn áp dụng trên toàn bộ cụm, bạn cần dùng `ClusterRole`.

---

### 2. Quy trình thiết lập chi tiết (Ví dụ thực tế)

Giả sử bạn có một ứng dụng trong namespace tên là `webapps` và nó cần quyền để liệt kê (list) các Pod trong đó.

#### Bước 1: Tạo Namespace và Service Account

Đầu tiên, chúng ta tạo môi trường và định danh cho ứng dụng:

```
# Tạo namespace
kubectl create namespace webapps

# Tạo Service Account tên là 'app-service-account'
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: webapps
EOF
```

#### Bước 2: Tạo Role (Danh sách quyền hạn)

Ở bước này, chúng ta định nghĩa các hành động (`verbs`) và tài nguyên (`resources`) mà Service Account được phép chạm vào.

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: webapps
rules:
- apiGroups: ["", "apps", "batch"] # Nhóm API (để trống "" là nhóm core)
  resources: ["pods", "deployments", "configmaps"] # Các loại tài nguyên
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"] # Các hành động được phép
```

_Lời khuyên:_ Trong thực tế, bạn **chỉ nên cấp những quyền thực sự cần thiết** thay vì cấp tất cả quyền như ví dụ trên để đảm bảo bảo mật.

#### Bước 3: Liên kết Role với Service Account (RoleBinding)

Lúc này, `app-role` và `app-service-account` vẫn chưa liên quan đến nhau. Chúng ta cần RoleBinding để "kích hoạt" quyền.

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-rolebinding
  namespace: webapps
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-role # Tên Role đã tạo ở bước 2
subjects:
- kind: ServiceAccount
  name: app-service-account # Tên Service Account đã tạo ở bước 1
  namespace: webapps
```

---

### 3. Cách sử dụng trong Pod hoặc Deployment

Để một Pod sử dụng quyền hạn này, bạn phải khai báo trường `serviceAccountName` trong file cấu hình của nó.

**Ví dụ với Deployment:**

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  template:
    spec:
      serviceAccountName: app-service-account # Chỉ định Service Account tại đây
      containers:
      - name: nginx
        image: nginx:1.14.2
```

---

### 4. Tại sao không dùng Service Account mặc định?

Mỗi namespace đều có một Service Account tên là `default`. Tuy nhiên, theo mặc định, nó **không có quyền truy cập vào API của K8s**. Nếu bạn cố gắng thực hiện lệnh kiểm tra tài nguyên từ một Pod dùng tài khoản mặc định, bạn sẽ nhận được lỗi: `Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:webapps:default" cannot list resource "pods"...`.

### 5. Kiểm tra và Xác nhận

Bạn có thể kiểm tra xem Service Account đã có quyền chưa bằng cách chạy một Pod có chứa công cụ `kubectl` và thử liệt kê các tài nguyên trong namespace đó:

1. Triển khai một Pod thử nghiệm (`debug pod`) với `serviceAccountName: app-service-account`.
2. Sử dụng lệnh `kubectl exec` để đi vào trong Pod đó.
3. Chạy lệnh `kubectl get pods` ngay bên trong Pod để xem kết quả.

**Tổng kết:** Việc quản lý quyền bằng Service Account giúp ứng dụng của bạn hoạt động mượt mà nhưng vẫn nằm trong tầm kiểm soát an toàn, tránh việc một ứng dụng bị xâm nhập có thể gây hại cho toàn bộ hệ thống K8s.