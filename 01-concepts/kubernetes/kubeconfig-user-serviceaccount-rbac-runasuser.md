---
doc_id: kubeconfig-user-serviceaccount-rbac-runasuser
title: Kubeconfig, User, ServiceAccount, RBAC và runAsUser trong Kubernetes
legacy_title: Kubeconfig, user, ServiceAccount, RBAC, UID và runAsUser trong Kubernetes
type: concept-note
domain: kubernetes
keywords:
  - kubeconfig
  - user
  - serviceaccount
  - rbac
  - authentication
  - authorization
  - runasuser
  - uid
  - securitycontext
  - openshift
  - scc
status: active
---

# Kubeconfig, User, ServiceAccount, RBAC và runAsUser trong Kubernetes

## Nguồn tham chiếu

- [Kubernetes Docs - Organizing Cluster Access Using kubeconfig Files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
- [Kubernetes Docs - Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
- [Kubernetes Docs - Authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
- [Kubernetes Docs - Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Kubernetes Docs - Service Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/)
- [Kubernetes Docs - Configure Service Accounts for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
- [Kubernetes Docs - Configure a Security Context for a Pod or Container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- [Kubernetes Docs - Object Names and IDs](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/)
- [Red Hat Docs - Managing security context constraints](https://docs.redhat.com/en/documentation/openshift_container_platform/4.8/html/authentication_and_authorization/managing-pod-security-policies)

## Kết luận 1 dòng

==`kubeconfig` giúp client đăng nhập vào cluster, `RBAC` quyết định quyền trên Kubernetes API, `ServiceAccount` là identity của app trong Pod, còn `runAsUser` là Linux UID của process trong container.==

## Bản đồ 2 lớp

### 1. API layer

Lớp này trả lời:

- ai đang gọi Kubernetes API
- identity đó được `get`, `list`, `create`, `delete` gì

Các khái niệm thuộc lớp này:

- `kubeconfig`
- human `user`
- `group`
- `ServiceAccount`
- `authentication`
- `authorization`
- `RBAC`

### 2. Runtime layer

Lớp này trả lời:

- process trong container chạy bằng UID nào
- process có quyền ghi file hay volume không
- container có đang chạy bằng `root` không

Các khái niệm thuộc lớp này:

- `securityContext`
- `runAsUser`
- `runAsGroup`
- `fsGroup`

**Câu chốt:**  
==`RBAC` là quyền với Kubernetes API, còn `runAsUser` là quyền Linux bên trong container.==

## Cách chúng hoạt động theo luồng

### A. Human user dùng `kubectl`

Luồng là:

1. `kubectl` đọc kubeconfig
2. gửi request lên API server với credential
3. API server authenticate ra username
4. RBAC kiểm tra user/group này có quyền gì

Ở đây subject của RBAC thường là **user** hoặc **group**. 

### B. Pod gọi Kubernetes API

Luồng là:

1. Pod có `serviceAccountName`
2. Pod được gắn credential của ServiceAccount
3. app trong Pod dùng credential đó gọi API server
4. API server authenticate ra username dạng `system:serviceaccount:<namespace>:<name>`
5. RBAC kiểm tra ServiceAccount đó có quyền gì

Kubernetes docs ghi rõ username của ServiceAccount có dạng `system:serviceaccount:(NAMESPACE):(SERVICEACCOUNT)`. 

### C. Container process chạy trong Pod

Luồng này khác hẳn:

1. Pod spec có thể có `securityContext.runAsUser`
2. process trong container chạy với UID Linux như 1000
3. UID này quyết định quyền file/process trong container
4. nó **không tự quyết định RBAC API permissions**

Đây là lớp OS/runtime, không phải lớp API identity.

![[Kubernetes authentication and authorization overview.png]]

![[Pasted image 20260412030629.png]]
Về bản chất, **human user** và **ServiceAccount** khác nhau ở chỗ này:

- **Human user**: thường là danh tính từ bên ngoài cluster, ví dụ client cert, bearer token, OIDC, LDAP/SSO proxy. Kubernetes tự nó **không có User API object** cho normal users. 
- **ServiceAccount**: là object thật trong Kubernetes API, có namespace, và được thiết kế làm identity cho **process chạy trong Pod** hoặc cho automation/workloads.
# VÍ DỤ

Ví dụ A: Dev trong công ty cần chỉ được xem Pod ở namespace dev

Đây là human user. Bạn thường:

- authenticate user từ OIDC/cert
- tạo `Role` trong namespace `dev`
- tạo `RoleBinding` bind role cho user hoặc group đó 

### Ví dụ B: Ứng dụng trong Pod cần đọc ConfigMap trong namespace `dev`

Đây là workload identity. Bạn thường:

- tạo `ServiceAccount`
- tạo `Role`
- tạo `RoleBinding`
- gán `serviceAccountName` cho Pod 

### Ví dụ C: Container phải chạy non-root

Đây không phải RBAC. Bạn dùng:

- `securityContext.runAsUser: 1000`  
    để process trong container chạy với UID 1000.

## Nhanh gọn để phân biệt

| Khái niệm | Thực chất là gì | Không phải gì |
|---|---|---|
| `kubeconfig` | file cấu hình để `kubectl` biết cluster nào, endpoint nào, credential nào | không phải nơi cấp quyền |
| human `user` | danh tính từ bên ngoài cluster, thường là người dùng `kubectl` | thường không phải object `User` để `kubectl get` |
| `ServiceAccount` | identity nội bộ cho workload trong Pod | không phải human user |
| `RBAC` | cơ chế phân quyền cho `user`, `group`, `ServiceAccount` | không phải cơ chế chạy process trong container |
| `Pod` | nơi workload chạy | không phải account để RBAC bind quyền |
| `metadata.uid` | mã định danh duy nhất của object | không phải user ID đăng nhập |
| `runAsUser` | Linux UID của process trong container | không phải quyền RBAC |

## 2 flow phải nắm

### Flow 1: human user dùng `kubectl`

1. `kubectl` đọc `kubeconfig`.
2. API server xác thực user hoặc group là ai.
3. Authorization, thường là `RBAC`, kiểm tra quyền.
4. Request thành công hoặc bị `Forbidden`.

**Câu chốt:**  
==`kubeconfig` giúp vào cluster, còn `RBAC` quyết định vào được tới đâu.==

### Flow 2: app trong Pod gọi Kubernetes API

1. Pod dùng `spec.serviceAccountName` hoặc rơi về `default` ServiceAccount.
2. App trong container dùng credential của `ServiceAccount` đó để gọi API server.
3. API server nhận ra identity dạng `system:serviceaccount:<namespace>:<name>`.
4. `RBAC` kiểm tra ServiceAccount đó có quyền hay không.

**Câu chốt:**  
==không phải Pod tự có quyền; app trong Pod dùng identity của `ServiceAccount` để gọi API.==

## Ví dụ gắn mọi thứ lại với nhau

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: reader
  namespace: dev
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: dev
subjects:
  - kind: ServiceAccount
    name: reader
    namespace: dev
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
---
apiVersion: v1
kind: Pod
metadata:
  name: app
  namespace: dev
spec:
  serviceAccountName: reader
  securityContext:
    runAsUser: 1000
  containers:
    - name: app
      image: nginx
```

Ví dụ này nói đúng 4 chuyện:

1. Pod `app` dùng `ServiceAccount` tên `reader`.
2. App trong Pod có thể `get/list pods` trong namespace `dev`.
3. App không có quyền `delete pods` vì `Role` không cấp quyền đó.
4. Process trong container chạy bằng Linux UID `1000`, nhưng UID này không làm RBAC mạnh hơn hay yếu đi.

## Những điều rất dễ nhầm

### `kubeconfig` không cấp quyền

Nó chỉ chứa:

- cluster nào
- endpoint nào
- credential nào

Quyền thật sự nằm ở authorization, thường là `RBAC`.

### `ServiceAccount` không phải human user

Human user thường là người dùng `kubectl` từ bên ngoài cluster.  
`ServiceAccount` là identity cho workload bên trong cluster.

### Pod không tự chọn `ServiceAccount` theo `RoleBinding`

Pod dùng:

- `spec.serviceAccountName`
- hoặc `default`

`RoleBinding` chỉ cấp quyền cho identity đó.

### `metadata.uid` không liên quan tới login hay RBAC

Đó chỉ là object ID để Kubernetes phân biệt resource này với resource khác.

### `runAsUser` không quyết định quyền trên API server

Bạn có thể đổi:

- `runAsUser: 1000`
- thành `runAsUser: 0`

thì quyền RBAC vẫn y nguyên nếu `ServiceAccount` không đổi.

## OpenShift là câu chuyện của lớp nào

OpenShift thường siết mạnh **runtime layer** bằng `SecurityContextConstraints`.

Điều đó nghĩa là:

- `RBAC` có thể đúng
- `ServiceAccount` có thể đúng
- nhưng Pod vẫn fail nếu `runAsUser` hoặc `securityContext` không phù hợp policy

**Câu chốt:**  
==Kubernetes API permission đúng chưa đủ; runtime security của container cũng phải đúng.==

## 6 câu học thuộc

1. `kubeconfig` là cấu hình kết nối của client, không phải nơi cấp quyền.
2. Human user là danh tính từ bên ngoài cluster.
3. `ServiceAccount` là identity của workload trong Pod.
4. `RBAC` quyết định identity đó được làm gì với Kubernetes API.
5. `runAsUser` là Linux UID của process trong container.
6. `metadata.uid` chỉ là mã định danh của object.
