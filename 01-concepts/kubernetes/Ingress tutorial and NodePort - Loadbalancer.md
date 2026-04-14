### 1. Kubernetes Ingress là gì? (Dành cho người mới)

Trong tiếng Anh, **Ingress** có nghĩa là "hành động đi vào". Trong K8s, Ingress là một tài nguyên cho phép bạn thiết lập các **quy tắc điều hướng lưu lượng truy cập** từ các nguồn bên ngoài (Internet) đến các ứng dụng đang chạy bên trong cụm.

**Tại sao cần Ingress?** Trước khi có Ingress, để đưa một ứng dụng ra ngoài, bạn thường phải dùng dịch vụ loại `LoadBalancer`. Tuy nhiên, cách này gây tốn kém và khó quản lý nếu bạn có hàng chục ứng dụng (mỗi ứng dụng lại cần một LoadBalancer riêng). Ingress giúp bạn sử dụng **một điểm vào duy nhất** nhưng có thể dẫn khách đến nhiều "phòng" (dịch vụ) khác nhau dựa trên tên miền hoặc đường dẫn,.

---

### 2. Hai thành phần không thể tách rời

Để Ingress hoạt động, bạn cần hiểu rõ hai khái niệm này:

- **Kubernetes Ingress Resource (Tài nguyên Ingress):** Đây là nơi bạn **viết các quy tắc** cấu hình DNS (ví dụ: tên miền nào trỏ về dịch vụ nào),. Nó chỉ là một file cấu hình được lưu trong hệ thống.
- **Kubernetes Ingress Controller (Bộ điều khiển Ingress):** Đây là **"động cơ" thực hiện việc điều hướng**. Nó không có sẵn mặc định khi bạn cài K8s. Bạn phải tự cài các bộ điều khiển như Nginx, Traefik, hay HAProxy,. Ingress Controller đóng vai trò như một máy chủ proxy ngược (Reverse Proxy).

---

### 3. Ví dụ chi tiết về File cấu hình (YAML)

Giả sử bạn có một dịch vụ tên là `hello-service` và bạn muốn người dùng truy cập qua tên miền `test.apps.example.com`.

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
  namespace: dev
spec:
  rules:
  - host: test.apps.example.com  # Tên miền bên ngoài
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-service  # Tên dịch vụ bên trong K8s
            port:
              number: 80         # Cổng của dịch vụ
```

**Giải thích:** Cấu hình này ra lệnh rằng: "Mọi yêu cầu gửi đến tên miền `test.apps.example.com` hãy chuyển hướng nó vào dịch vụ `hello-service` ở cổng 80 trong namespace `dev`",.

---

### 4. Cách thức hoạt động (Luồng đi của dữ liệu)

Hãy lấy ví dụ với bộ điều khiển phổ biến nhất là **Nginx Ingress Controller**:

1. **Lắng nghe:** Nginx Controller liên tục theo dõi API của K8s để xem bạn có tạo hoặc cập nhật Ingress Resource nào không.
2. **Cập nhật cấu hình:** Khi thấy có quy tắc mới (như file YAML ở trên), Nginx sẽ tự động tạo file cấu hình tương ứng bên trong Pod của nó (tại `/etc/nginx/conf.d`).
3. **Thực thi:** Khi có người dùng truy cập từ Internet, yêu cầu sẽ đi qua: **Load Balancer bên ngoài -> Ingress Controller -> Service bên trong K8s -> Pod ứng dụng**,.

---

### 5. Các tính năng nâng cao

- **Điều hướng theo đường dẫn (Path-based):** Bạn có thể cấu hình để `vidu.com/login` trỏ về ứng dụng A, còn `vidu.com/pay` trỏ về ứng dụng B trong cùng một Ingress,.
- **Hỗ trợ TLS/SSL:** Ingress cho phép bạn cấu hình bảo mật HTTPS bằng cách sử dụng các chứng chỉ số được lưu trong K8s Secret,.
- **Đa dạng lựa chọn:** Có nhiều loại Ingress Controller tùy theo môi trường bạn dùng như GKE Ingress (cho Google Cloud), AWS ALB Ingress (cho Amazon), hoặc Nginx cho mọi môi trường.

---

### 6. Câu hỏi thường gặp (FAQ) cho người mới

- **Ingress có phải là Load Balancer không?** Không. Ingress chứa các quy tắc điều hướng, còn **Ingress Controller mới đóng vai trò là Load Balancer**.
- **Tạo quy tắc Ingress xong là chạy được ngay?** Chưa chắc. Bạn phải đảm bảo đã cài đặt một Ingress Controller trong cụm, nếu không các quy tắc bạn viết sẽ không có tác dụng gì,.

Dưới đây là bản tổng hợp theo kiểu **lý thuyết ghi chú**, trình bày lại mạch khái niệm cho dễ học và dễ nhớ.

---

# ==Service, NodePort, LoadBalancer và Ingress trong Kubernetes==

## 1. Bài toán gốc: làm sao request từ bên ngoài đi vào được ứng dụng trong cluster

Một ứng dụng chạy trong Kubernetes thường không được truy cập trực tiếp từ Internet hay từ mạng bên ngoài ngay lập tức.  
Lý do là Pod có IP nội bộ, thay đổi được, và không phải đối tượng để người dùng bên ngoài kết nối trực tiếp.

Vì vậy, muốn truy cập ứng dụng từ bên ngoài, cần có **một cơ chế đưa traffic vào cluster**.  
Sau khi traffic đã vào cluster, mới cần tiếp tục **một cơ chế định tuyến đúng đến ứng dụng mong muốn**.

Từ đây xuất hiện 2 lớp chức năng khác nhau:

- **Lớp đưa traffic vào cluster**
    
- **Lớp định tuyến traffic bên trong cluster**
    

Đây là điểm mấu chốt để phân biệt các khái niệm thường bị lẫn.

---

# 2. Service trong Kubernetes

## 2.1. Vai trò của Service

Service là đối tượng cung cấp một địa chỉ ổn định để truy cập một nhóm Pod.  
Pod có thể thay đổi, scale lên xuống, restart, nhưng Service vẫn giữ vai trò là điểm truy cập ổn định.

Nói đơn giản:

- **Pod** là nơi chạy ứng dụng thật
    
- **Service** là lớp đứng trước Pod để gom các Pod lại và phân phối request tới Pod phù hợp
    

Ví dụ:

- Có 3 Pod backend
    
- Service `backend-service` đứng trước 3 Pod đó
    
- Khi request đi vào `backend-service`, Kubernetes sẽ chọn 1 Pod phù hợp để xử lý
    

---

## 2.2. Ý nghĩa của Service

Service giải quyết bài toán:

- không cần biết Pod nào đang sống
    
- không cần gọi trực tiếp IP Pod
    
- ứng dụng khác trong cluster chỉ cần gọi tên Service
    

Ví dụ nội bộ cluster:

- frontend gọi `http://backend-service`
    
- không cần biết backend đang là Pod A, B hay C
    

---

# 3. NodePort

## 3.1. Khái niệm

NodePort là một kiểu Service cho phép mở một cổng trên **mọi node** trong cluster.  
Khi đó, từ bên ngoài có thể truy cập vào ứng dụng bằng:

`IP_NODE:NodePort`

Ví dụ:

- Service được mở bằng NodePort `30080`
    
- Cluster có 3 node:
    
    - `10.0.0.11`
        
    - `10.0.0.12`
        
    - `10.0.0.13`
        

Thì người dùng có thể gọi:

- `10.0.0.11:30080`
    
- `10.0.0.12:30080`
    
- `10.0.0.13:30080`
    

và đều có thể đi vào cùng một ứng dụng.

---

## 3.2. Bản chất của NodePort

NodePort là **cách đưa traffic từ ngoài vào cluster bằng một port cố định trên node**.

Nó chỉ trả lời câu hỏi:

**“Muốn đi từ bên ngoài vào cluster thì đi qua IP và port nào?”**

NodePort **không quyết định ứng dụng nào sẽ nhận request theo domain hay theo path**.  
Nó chỉ là **cửa đi vào**.

---

## 3.3. Ví dụ thực tế

Giả sử có ứng dụng `my-app` được expose bằng NodePort `30080`.

Luồng request:

`Client -> 10.0.0.11:30080 -> Service -> Pod`

Hoặc:

`Client -> 10.0.0.12:30080 -> Service -> Pod`

Điểm quan trọng là NodePort thường được dùng khi:

- lab
    
- môi trường test
    
- on-prem đơn giản
    
- chưa có LoadBalancer ngoài
    

---

## 3.4. Hạn chế của NodePort

NodePort hoạt động được nhưng có một số hạn chế:

- Phải dùng port cao, thường dạng `30000-32767`
    
- Không đẹp như port 80/443
    
- Nếu có nhiều ứng dụng thì phải mở nhiều port khác nhau
    
- Người dùng dễ phải nhớ port, ví dụ `app.company.com:30080`
    
- Không thuận tiện để quản lý nhiều ứng dụng theo domain/path
    

---

# 4. LoadBalancer

## 4.1. Khái niệm

LoadBalancer là kiểu Service giúp ứng dụng có thể được truy cập từ bên ngoài thông qua một **địa chỉ IP ngoài** ổn định, thường do cloud provider hoặc hệ thống load balancing cung cấp.

Bản chất của LoadBalancer vẫn là:

**một cách đưa traffic từ ngoài vào cluster**

Nó không thay thế Service, cũng không thay thế Ingress.  
Nó chỉ là một cơ chế expose ra ngoài tiện lợi và chuyên nghiệp hơn NodePort.

---

## 4.2. Vai trò thực tế

LoadBalancer thường được dùng để:

- cấp một IP public hoặc VIP ổn định
    
- đưa traffic vào cluster
    
- phân phối tải tới backend/node phù hợp
    
- hỗ trợ HA tốt hơn
    
- giảm phụ thuộc vào một node cụ thể
    

---

## 4.3. Ví dụ thực tế

Trên cloud, nếu một Service có kiểu `LoadBalancer`, hệ thống có thể cấp cho nó một IP public như:

`35.10.20.30`

Lúc này người dùng gọi:

`http://35.10.20.30`

và request sẽ được đưa vào cluster.

Nếu gắn DNS:

`app.company.com -> 35.10.20.30`

thì người dùng sẽ gọi:

`http://app.company.com`

Nhưng cần nhớ:  
**domain chỉ trỏ đến IP**.  
Chính LoadBalancer mới là thành phần đang nghe ở IP đó và nhận traffic thật.

---

# 5. Ingress

## 5.1. Khái niệm

Ingress là đối tượng dùng để khai báo **quy tắc định tuyến HTTP/HTTPS** vào các Service trong cluster.

Ingress không phải là một địa chỉ IP.  
Ingress cũng không tự mở cổng ra Internet.  
Ingress chỉ là **bộ luật định tuyến**.

Nó trả lời câu hỏi:

**“Khi request đã vào cluster rồi, thì nên chuyển đến service nào?”**

---

## 5.2. Ingress xử lý gì

Ingress thường định tuyến dựa trên:

- **domain/host**
    
- **path**
    

Ví dụ:

- `app.company.com` -> `frontend-service`
    
- `api.company.com` -> `api-service`
    
- `app.company.com/admin` -> `admin-service`
    

Như vậy, Ingress đặc biệt phù hợp với các ứng dụng web, microservices, API gateway đơn giản.

---

## 5.3. Bản chất của Ingress

Ingress không phải “cửa vào vật lý”.  
Nó giống như **bảng chỉ đường**.

Nói cách khác:

- NodePort / LoadBalancer: mở đường để xe đi vào khu vực cluster
    
- Ingress: chỉ xe đó nên đi đến tòa nhà nào trong cluster
    

---

# 6. Ingress Controller

## 6.1. Khái niệm

Ingress Controller là thành phần thật sự đọc Ingress rule và thực thi chúng.

Nếu chỉ tạo Ingress resource mà không có Ingress Controller, thì sẽ không có gì xảy ra.

Điều này rất quan trọng.

- **Ingress** = luật
    
- **Ingress Controller** = người thực hiện luật
    

Ví dụ phổ biến:

- ingress-nginx
    
- traefik
    
- haproxy ingress
    

---

## 6.2. Vai trò

Ingress Controller nhận request HTTP/HTTPS từ bên ngoài, sau đó:

1. đọc host/path
    
2. so khớp với rule Ingress
    
3. chuyển tiếp tới đúng Service
    
4. Service chuyển tiếp tới Pod
    

---

# 7. Quan hệ giữa NodePort, LoadBalancer và Ingress

## 7.1. Điểm dễ nhầm nhất

Nhiều người tưởng rằng:

`domain -> Ingress`

Thực ra chính xác phải là:

`domain -> IP`

Sau đó IP đó thuộc về một thành phần đang nghe traffic, ví dụ:

- NodePort trên node
    
- LoadBalancer
    
- hostNetwork
    
- edge proxy ngoài cluster
    

Rồi từ đó mới tới Ingress Controller.

---

## 7.2. Luồng đúng về mặt kiến trúc

### Trường hợp dùng NodePort cho ingress controller

`domain -> node IP:nodePort -> ingress controller -> service -> pod`

### Trường hợp dùng LoadBalancer cho ingress controller

`domain -> load balancer IP -> ingress controller -> service -> pod`

Cả hai trường hợp đều đúng.  
Khác nhau ở chỗ **traffic đi vào cluster bằng gì**.

---

# 8. Ý nghĩa của Ingress khi đã có NodePort

Đây là ý rất quan trọng.

Nếu đã có NodePort, tại sao còn cần Ingress?

Câu trả lời là:

- **NodePort** lo chuyện “đi vào cluster bằng cổng nào”
    
- **Ingress** lo chuyện “vào rồi thì request thuộc app nào”
    

---

## 8.1. Không có Ingress

Nếu không dùng Ingress, mỗi ứng dụng phải có một cổng riêng:

- app1 -> `30001`
    
- app2 -> `30002`
    
- app3 -> `30003`
    

Người dùng phải gọi theo từng port khác nhau.  
Quản lý phức tạp, khó nhớ, khó mở rộng.

---

## 8.2. Có Ingress

Chỉ cần expose **Ingress Controller** bằng một NodePort chung, ví dụ:

- HTTP -> `30100`
    
- HTTPS -> `30101`
    

Sau đó toàn bộ ứng dụng đi sau Ingress:

- `app.company.com:30100` -> frontend
    
- `api.company.com:30100` -> backend
    
- `admin.company.com:30100` -> admin
    

Như vậy:

- không cần mỗi app một port riêng
    
- chỉ cần một điểm vào chung
    
- route theo domain/path rất gọn
    

---

# 9. Domain và DNS không thay thế được NodePort hay LoadBalancer

## 9.1. Vai trò của DNS

DNS chỉ làm việc này:

`app.company.com -> 10.0.0.11`  
hoặc  
`app.company.com -> 35.10.20.30`

DNS chỉ trả lời:  
**“Tên miền này trỏ tới IP nào?”**

DNS không làm các việc sau:

- không mở cổng 80/443
    
- không chuyển traffic vào Pod
    
- không thực hiện rule host/path
    
- không cân bằng tải trong cluster
    

---

## 9.2. Ý nghĩa thực tế

Khi người dùng gõ `app.company.com`, điều xảy ra trước tiên là:

1. hệ thống tra DNS
    
2. lấy ra IP
    
3. gửi request đến IP đó
    

Do đó, muốn ứng dụng chạy được thì ở IP đó phải có một thành phần đang nghe, ví dụ:

- LoadBalancer
    
- NodePort trên node
    
- ingress controller bind trực tiếp bằng hostNetwork
    
- reverse proxy ngoài cluster
    

Vì vậy, chỉ “add domain vào Ingress” chưa đủ theo nghĩa vật lý mạng.  
Nó chỉ có tác dụng khi phía dưới đã có đường vào cluster sẵn.

---

# 10. Use case thực tế

## 10.1. Use case 1: Một app đơn giản, môi trường lab

Có một app nội bộ, ít người dùng, chỉ để test.

Giải pháp:

- expose app bằng NodePort
    
- truy cập bằng `10.0.0.11:30080`
    

Ưu điểm:

- đơn giản
    
- dễ dựng
    
- nhanh
    

Nhược điểm:

- phải nhớ port
    
- không đẹp
    
- không phù hợp khi nhiều app
    

---

## 10.2. Use case 2: Nhiều app web trong cùng cluster

Có 3 ứng dụng:

- `app.company.com`
    
- `api.company.com`
    
- `admin.company.com`
    

Giải pháp tốt hơn:

- deploy Ingress Controller
    
- expose Ingress Controller bằng NodePort hoặc LoadBalancer
    
- khai báo Ingress rule theo host
    

Luồng:

`domain -> điểm vào cluster -> ingress controller -> đúng service -> đúng pod`

Lợi ích:

- một điểm vào chung
    
- route theo domain/path
    
- dễ quản lý
    
- thuận tiện TLS
    

---

## 10.3. Use case 3: On-prem production

Cluster có 3 node, cần độ ổn định cao.

Giải pháp:

- đặt một LB ngoài hoặc dùng giải pháp tương đương
    
- LB đưa traffic vào ingress controller
    
- ingress controller route tới service
    

Lợi ích:

- có IP/VIP ổn định
    
- không phụ thuộc 1 node duy nhất
    
- dễ HA hơn
    
- dễ tổ chức nhiều ứng dụng dùng chung cổng 80/443
    

---

# 11. Kết luận tổng hợp

## 11.1. Phân biệt ngắn gọn

- **Service**: lớp ổn định đứng trước Pod
    
- **NodePort**: mở cổng trên node để traffic từ ngoài đi vào cluster
    
- **LoadBalancer**: một cách expose ra ngoài bằng IP ngoài ổn định
    
- **Ingress**: bộ luật route HTTP/HTTPS theo host/path
    
- **Ingress Controller**: thành phần thực thi luật Ingress
    

---

## 11.2. Câu nhớ nhanh

**NodePort / LoadBalancer trả lời:**  
“Traffic từ ngoài sẽ đi vào cluster bằng đâu?”

**Ingress trả lời:**  
“Sau khi đã vào cluster, request này phải đi tới service nào?”

---

## 11.3. Công thức ghi nhớ cuối cùng

### Không dùng Ingress

`Client -> NodePort/LoadBalancer -> Service -> Pod`

### Dùng Ingress

`Client -> NodePort/LoadBalancer -> Ingress Controller -> Service -> Pod`

---

Nếu bạn muốn, tôi có thể viết tiếp cho bạn một bản **“so sánh dạng bảng”** giữa `ClusterIP / NodePort / LoadBalancer / Ingress`để bạn chép note nhanh hơn.