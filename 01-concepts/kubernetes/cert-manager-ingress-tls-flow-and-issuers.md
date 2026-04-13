---
doc_id: cert-manager-ingress-tls-flow-and-issuers
title: cert-manager, Ingress và quy trình cấp TLS trong Kubernetes
legacy_title: Kubernetes - cert manager - secure pipeline process..
type: concept-note
domain: kubernetes
keywords:
  - cert-manager
  - ingress
  - ingress-controller
  - tls
  - certificate
  - issuer
  - clusterissuer
  - acme
  - lets-encrypt
  - http01
  - dns01
status: active
---

# cert-manager, Ingress và quy trình cấp TLS trong Kubernetes

Nguồn tham khảo:
- [DevOpsCube - Nginx Ingress with Cert Manager and Letsencrypt](https://devopscube.com/nginx-ingress-with-cert-manager/)
- [cert-manager Docs - Issuer](https://cert-manager.io/docs/concepts/issuer/)
- [cert-manager Docs - Annotated Ingress resource](https://cert-manager.io/docs/usage/ingress/)
- [cert-manager Docs - ACME](https://cert-manager.io/docs/configuration/acme/)
- [cert-manager Docs - ACME Orders and Challenges](https://cert-manager.io/docs/concepts/acme-orders-challenges/)
- [cert-manager Docs - Helm](https://cert-manager.io/docs/installation/helm/)
- [Kubernetes Docs - Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Let's Encrypt Docs - Challenge Types](https://letsencrypt.org/docs/challenge-types/)
- [Let's Encrypt Docs - Staging Environment](https://letsencrypt.org/docs/staging-environment/)
- [Let's Encrypt Docs - Profiles](https://letsencrypt.org/docs/profiles/)

Ghi chú cập nhật:
- ==Nội dung đã được đối chiếu với tài liệu chính thức tính đến `2026-04-12`.==
- ==`Ingress` API vẫn tồn tại nhưng đã bị `frozen`; Kubernetes khuyến nghị `Gateway API` cho thiết kế mới.==
- ==cert-manager docs hiện khuyến nghị cài bằng OCI chart `oci://quay.io/jetstack/charts/cert-manager`.==
- ==cert-manager docs latest có cảnh báo về lộ trình end-of-life của `ingress-nginx` trong mốc `2026-03`, nên với hệ thống mới cần theo dõi kế hoạch migration.==

## Kết luận ngắn gọn trước

==`Ingress` chỉ là object mô tả route HTTP/HTTPS; nó không tự proxy traffic.==

==`ingress controller` mới là thành phần thật sự nhận traffic từ bên ngoài và dùng TLS Secret để terminate HTTPS.==

==`cert-manager` không đi nhận traffic; nó quản lý vòng đời certificate và ghi `certificate + private key` vào `Secret`.==

==Khi annotate `Ingress`, `ingress-shim` của cert-manager sẽ đảm bảo có `Certificate`, rồi từ đó sinh `CertificateRequest -> Order -> Challenge -> Secret`.==

==`Issuer` chỉ dùng trong một namespace; `ClusterIssuer` dùng được toàn cluster.==

==Với cert public qua ACME, `HTTP01` hợp cho host thường; `DNS01` gần như bắt buộc nếu cần wildcard.==

## Bức tranh lớn: bài toán đang giải là gì?

Bài toán gốc không phải chỉ là "cài cert-manager", mà là làm cho một ứng dụng trong Kubernetes đi hết chuỗi:

1. Pod chạy ứng dụng.
2. Service expose Pod trong nội bộ cluster.
3. Ingress khai báo host/path từ Internet đi vào Service nào.
4. ingress controller đọc Ingress và thật sự phục vụ traffic.
5. cert-manager xin certificate cho host public đó.
6. Certificate được lưu vào TLS Secret.
7. ingress controller dùng Secret đó để terminate HTTPS.

Ví dụ đời thường:

- `Pod` là cửa hàng.
- `Service` là số nội bộ để gọi đúng cửa hàng.
- `Ingress` là bảng chỉ đường.
- `ingress controller` là người thật đứng ngoài đường và phân luồng.
- `cert-manager` là bộ phận đi làm và gia hạn "giấy chứng nhận HTTPS".
- `Let's Encrypt` là CA public cấp chứng chỉ trong use case phổ biến nhất.

## Hiểu đúng vai trò từng thành phần

### 1. Ingress

`Ingress` là Kubernetes API object để mô tả cách HTTP/HTTPS từ bên ngoài đi vào các Service trong cluster.

Điểm quan trọng:

- ==`Ingress` chỉ là khai báo desired state.==
- nó không tự mở cổng, không tự proxy và không tự terminate TLS
- muốn `Ingress` hoạt động thì cluster phải có ít nhất một `ingress controller`

Kubernetes hiện ghi rất rõ:

- ==`Ingress` API đã bị `frozen`.==
- ==Kubernetes khuyến nghị dùng `Gateway API` cho thiết kế mới.==

### 2. ingress controller

`ingress controller` là thành phần thật sự watch `Ingress` rồi cấu hình reverse proxy/load balancer tương ứng.

Điểm cần nhớ:

- cluster có thể có nhiều ingress controller
- vì vậy nên chỉ rõ `ingressClassName`
- nếu không chỉ rõ class, bạn có thể vô tình để sai controller nhận resource

### 3. cert-manager

`cert-manager` là certificate controller cho Kubernetes/OpenShift.

Nó làm những việc chính:

- tạo certificate
- gia hạn certificate
- làm việc với nhiều nguồn ký khác nhau
- lưu kết quả vào `Secret`
- cập nhật trạng thái các resource liên quan tới issuance

### 4. Certificate

`Certificate` là resource mô tả:

- muốn xin cert cho tên nào
- dùng issuer nào
- lưu vào Secret nào

Nếu bạn dùng Ingress annotation, nhiều khi bạn không tạo `Certificate` bằng tay; `ingress-shim` sẽ tạo thay bạn.

### 5. Secret TLS

Secret đích mà cert-manager tạo ra thường là kiểu `kubernetes.io/tls`.

Đây là thứ ingress controller thật sự cần để bắt tay TLS với client.

### 6. Issuer và ClusterIssuer

`Issuer`/`ClusterIssuer` là resources đại diện cho CA hoặc cách ký certificate.

- `Issuer`: namespaced, chỉ dùng trong cùng namespace
- `ClusterIssuer`: cluster-scoped, dùng được từ nhiều namespace

Một nuance hay bị quên:

- với `ClusterIssuer`, các Secret cấu hình mà nó tham chiếu thường được tìm trong `Cluster Resource Namespace`
- mặc định namespace đó là `cert-manager`
- controller flag có thể đổi namespace này

## Luồng tài nguyên thật sự khi dùng Ingress + cert-manager

Đây là chỗ dễ hiểu sai nhất.

<span style="color:red">Không phải ingress controller đi xin certificate.</span>

Luồng đúng là:

1. Bạn tạo `Ingress` có annotation như `cert-manager.io/cluster-issuer`.
2. `ingress-shim` của cert-manager thấy Ingress đó.
3. `ingress-shim` đảm bảo tồn tại một `Certificate` tương ứng với `tls.secretName`.
4. cert-manager tạo `CertificateRequest` từ `Certificate`.
5. Nếu issuer là `ACME`, cert-manager tạo tiếp `Order`.
6. Từ `Order`, cert-manager tạo một hoặc nhiều `Challenge`.
7. cert-manager solve challenge bằng `HTTP01` hoặc `DNS01`.
8. Khi CA xác minh thành công, cert-manager ghi `tls.crt` và `tls.key` vào `Secret`.
9. ingress controller dùng Secret đó để phục vụ HTTPS.

Tài nguyên theo pipeline:

```text
Ingress
  -> Certificate
    -> CertificateRequest
      -> Order
        -> Challenge
          -> Secret
            -> ingress controller dùng để terminate TLS
```

Điểm cực quan trọng:

- ==`Order` và `Challenge` là resource nội bộ để debug, không phải thứ end-user thường tạo tay.==
- ==Nếu cert không ra, hãy xem `Certificate`, `CertificateRequest`, `Order`, `Challenge` theo đúng thứ tự này.==

## Ví dụ manifest tối thiểu để hình dung flow

### 1. `ClusterIssuer` dùng Let's Encrypt staging qua `HTTP01`

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-http01-staging
spec:
  acme:
    email: ops@example.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-http01-staging-account-key
    solvers:
      - http01:
          ingress:
            ingressClassName: nginx
```

### 2. `Ingress` tham chiếu issuer đó

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: demo
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-http01-staging
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-svc
                port:
                  number: 80
  tls:
    - hosts:
        - app.example.com
      secretName: app-example-com-tls
```

Ý nghĩa của ví dụ:

- `tls.secretName` là nơi cert-manager sẽ lưu cert thật
- `Ingress` không tự tạo cert; nó chỉ kích hoạt `ingress-shim` tạo `Certificate`
- nên dùng endpoint `staging` trước, test flow xong rồi mới đổi sang production

## Issuer và ClusterIssuer: chọn cái nào?

| Tiêu chí | `Issuer` | `ClusterIssuer` |
|---|---|---|
| Scope | trong 1 namespace | toàn cluster |
| Có dùng chéo namespace được không | không | có |
| Phù hợp khi nào | team/service riêng, scope hẹp | nhiều namespace cùng dùng một CA/ACME config |
| Hay dùng cho public HTTPS | có thể | thường gặp hơn |
| Dễ quản trị tập trung | kém hơn | tốt hơn |

Ví dụ:

- `team-a` có CA nội bộ riêng cho namespace của mình -> dùng `Issuer`
- nhiều app ở nhiều namespace cùng xin cert public từ Let's Encrypt -> thường dùng `ClusterIssuer`

## Các issuer built-in quan trọng của cert-manager

Theo docs cấu hình và API hiện tại, phần built-in cần nắm nhất là 5 nhóm sau:

| Loại | Ký cert từ đâu | Use case điển hình | Có public trust mặc định không |
|---|---|---|---|
| `ACME` | ACME CA như Let's Encrypt hoặc ACME-compatible CA | website/app public, automation mạnh | thường có nếu CA là public |
| `CA` | CA cert + private key bạn tự giữ trong Secret | internal PKI, mTLS nội bộ | không |
| `Vault` | HashiCorp Vault PKI backend | enterprise PKI, policy/audit tốt | thường là không |
| `SelfSigned` | cert tự ký bằng chính key của nó | lab, bootstrap root CA | không |
| `CyberArk` | CyberArk Certificate Manager; trong API field lịch sử vẫn là `venafi` | enterprise certificate management | tùy hệ phía sau |

### 1. ACME issuer

Đây là loại phổ biến nhất cho HTTPS public.

Bạn thường cấu hình:

- `email`
- `server`
- `privateKeySecretRef`
- `solvers`

Điểm cần hiểu đúng:

- `privateKeySecretRef` ở đây là key của **ACME account**, không phải private key của website
- website cert cuối cùng sẽ nằm trong Secret của `Certificate`, không nằm trong account secret này

### 2. CA issuer

Dùng một CA cert/key có sẵn trong Secret để ký các cert khác.

Phù hợp khi:

- bạn tự vận hành internal PKI
- cần mTLS nội bộ
- không muốn phụ thuộc CA public

### 3. Vault issuer

Dùng Vault PKI backend để ký cert.

Phù hợp khi:

- doanh nghiệp đã có Vault làm trung tâm secrets/PKI
- cần audit, auth, policy rõ ràng

### 4. SelfSigned issuer

Tự ký cert bằng chính private key của cert đó.

Phù hợp khi:

- lab
- test nhanh
- bootstrap root CA ban đầu

Điểm cần nhớ:

- ==`SelfSigned` không phải lựa chọn chuẩn cho website public.==

### 5. CyberArk / `venafi`

Docs mới gọi theo CyberArk Certificate Manager, nhưng API field lịch sử vẫn là `venafi`.

Phù hợp khi:

- doanh nghiệp đã có nền tảng certificate management tập trung
- Kubernetes phải đi theo policy zone và governance có sẵn

## External issuer là gì?

Ngoài built-in issuer, cert-manager còn hỗ trợ external issuer.

Bản chất:

- controller đó nằm ngoài repo chính của cert-manager
- vẫn watch `CertificateRequest`
- vẫn hoạt động như một issuer bình thường
- chỉ khác ở `group` và controller implementation

Vì external issuer phụ thuộc hệ sinh thái và vendor, danh sách này thay đổi theo thời điểm. Khi học nền tảng, nên nắm chắc built-in trước rồi mới mở rộng sang external.

## ACME: phần dễ bị nhầm nhất

`ACME` không đồng nghĩa với `Let's Encrypt`.

Hiểu đúng là:

- `ACME` là giao thức
- `Let's Encrypt` là một CA public triển khai ACME
- cert-manager có `ACME issuer`
- ACME issuer có thể trỏ tới Let's Encrypt hoặc một ACME server khác

## cert-manager solve ACME challenge như thế nào?

Docs hiện tại của cert-manager ghi rõ built-in ACME flow dùng 2 loại challenge chính:

1. `HTTP01`
2. `DNS01`

### Bảng so sánh nhanh

| Loại | ACME CA kiểm tra cái gì | Wildcard được không | Hợp với tình huống nào |
|---|---|---|---|
| `HTTP01` | URL HTTP dưới `/.well-known/acme-challenge/...` | không | website public có ingress/load balancer public |
| `DNS01` | TXT record `_acme-challenge.<domain>` | có | wildcard, private web backend, multi-ingress, DNS automation tốt |

### HTTP01

Cách hoạt động:

- ACME CA truy cập URL HTTP trên chính domain đó
- cert-manager tự cấu hình route challenge tới một solver nhỏ
- domain phải resolve public và truy cập được từ Internet

Điểm rất quan trọng:

- ==`HTTP01` không dùng được cho wildcard certificate.==
- ==`HTTP01` phụ thuộc chuyện domain đã trỏ đúng tới ingress/load balancer.==

Ví dụ hợp lý:

- `app.example.com`
- cluster có ingress-nginx public
- DNS public trỏ đúng vào external load balancer

### DNS01

Cách hoạt động:

- cert-manager tạo hoặc yêu cầu bạn tạo TXT record `_acme-challenge.<domain>`
- ACME CA query DNS để kiểm tra
- khi DNS propagate xong thì challenge pass

Điểm cần nhớ:

- ==`DNS01` dùng được cho wildcard.==
- ==`DNS01` không dùng để chứng minh quyền sở hữu IP address.==
- nếu bạn muốn automation thật sự, DNS provider nên có API

Ví dụ hợp lý:

- `*.example.com`
- nhiều subdomain động
- service không muốn expose challenge HTTP ra Internet
- DNS đang ở Route53 / Cloudflare / Azure DNS / Google Cloud DNS

### TLS-ALPN-01 thì sao?

Let's Encrypt vẫn có challenge type `TLS-ALPN-01`.

Nhưng cần phân biệt:

- challenge này thuộc ACME/Let's Encrypt nói chung
- docs cert-manager hiện nêu built-in challenge flow ở mức thực tế là `HTTP01` và `DNS01`

Vì vậy khi học cert-manager, trọng tâm của bạn nên là `HTTP01` và `DNS01`.

## Route53 trong tutorial gốc đang đóng vai trò gì?

Đây là điểm rất dễ bị hiểu sai.

Nếu tutorial dùng:

- `Let's Encrypt`
- `ACME`
- `HTTP01`
- domain trỏ qua Route53 vào ELB/load balancer

thì Route53 ở đây thường đang làm việc:

- tạo record public để domain đi đúng vào ingress/load balancer

Nó **không tự động có nghĩa** là bạn đang dùng `DNS01`.

Chỉ khi cert-manager thật sự solve bằng TXT record `_acme-challenge...` thì lúc đó mới là `DNS01`.

## Let's Encrypt: những phần quan trọng nhất cần nhớ

### 1. Production vs Staging

| Môi trường | Endpoint | Dùng để làm gì | Browser trust |
|---|---|---|---|
| Production | `https://acme-v02.api.letsencrypt.org/directory` | cert thật | có |
| Staging | `https://acme-staging-v02.api.letsencrypt.org/directory` | test flow trước khi sang prod | không |

Điểm cần nhớ:

- ==nên test bằng `staging` trước==
- ==ACME account của staging và production là tách nhau==
- staging giúp tránh đụng rate limit production khi bạn còn đang sửa YAML hoặc DNS

### 2. Challenge types của Let's Encrypt

Let's Encrypt hiện nêu 3 challenge type chính:

- `HTTP-01`
- `DNS-01`
- `TLS-ALPN-01`

Trong đó:

- `HTTP-01` và `TLS-ALPN-01` có thể dùng để validate IP addresses
- `DNS-01` dùng được cho wildcard
- `TLS-ALPN-01` không phải hướng phổ biến cho đa số người dùng Kubernetes

### 3. Certificate profiles

Tính đến `2026-04-12`, Let's Encrypt có phần profile là điểm mới khá quan trọng.

Điểm cần hiểu đúng trước:

- không phải mọi environment đều quảng bá cùng một danh sách profile
- không phải mọi ACME client đều hỗ trợ chọn profile
- danh sách profile khả dụng thật sự phải xem từ `directory` endpoint của environment đang dùng

Các profile đáng nhớ:

| Profile | Ý chính | Validity | Khi nào đáng cân nhắc |
|---|---|---:|---|
| `classic` | mặc định, hành vi truyền thống | 90 ngày | muốn ổn định, ít thay đổi |
| `tlsserver` | cert gọn hơn, thiên về automation hiện đại | 90 ngày | web/server auth hiện đại |
| `shortlived` | giống `tlsserver` nhưng rất ngắn hạn | 160 giờ | automation renew cực chắc |
| `tlsclient` | có TLS Client Auth EKU | 90 ngày | chỉ khi thật sự cần client auth, và đây không phải hướng dài hạn |

Điểm cần nhớ:

- cert-manager ACME có field `profile`
- nếu không chỉ định, với Let's Encrypt mặc định là `classic`
- `shortlived` chỉ hợp khi bạn thực sự tin tưởng automation renew
- theo trang Profiles hiện tại, `tlsclient` là profile chuyển tiếp và có mốc ngừng hỗ trợ:
  - đã dùng trước `2026-05-13` thì còn dùng tới `2026-07-08`
  - nếu không thực sự cần client auth thì nên tránh

### 4. Wildcard và IP certificate

- wildcard certificate có được, nhưng ==bắt buộc `DNS01`==
- IP certificate là chủ đề mới hơn và không phải use case cert-manager/Ingress phổ biến nhất
- nếu mục tiêu chính là HTTPS cho website/app public trên Kubernetes, trọng tâm học vẫn nên là:
  - host name
  - `HTTP01` vs `DNS01`
  - `Issuer` vs `ClusterIssuer`
  - `Secret` và `Certificate` flow

## Các điểm cập nhật quan trọng so với tutorial cũ

### 1. Cài cert-manager bằng Helm

Docs hiện tại khuyến nghị:

- ưu tiên OCI chart `oci://quay.io/jetstack/charts/cert-manager`
- bật `--set crds.enabled=true`
- ==không embed cert-manager làm subchart một cách tùy tiện==
- ==cert-manager phải được cài đúng một lần trong cluster==

### 2. `ingressClassName` tốt hơn annotation class cũ

Với Ingress và ACME HTTP01, tư duy hiện đại là ưu tiên `ingressClassName`.

`class: nginx` vẫn là thứ bạn còn gặp trong tutorial cũ, nhưng khi viết mới thì nên nghiêng về `ingressClassName`.

### 3. `ingress-nginx` và `Ingress` không còn là câu chuyện "đương nhiên mãi mãi"

Tính đến `2026-04-12`:

- Kubernetes docs vẫn giữ `Ingress`, nhưng API này đã frozen
- cert-manager docs latest có cảnh báo về lộ trình end-of-life của `ingress-nginx` trong mốc `2026-03`

Điều này không làm kiến thức về Ingress trở nên vô ích.

Ngược lại:

- ==hiểu Ingress + cert-manager vẫn rất quan trọng để nắm nền tảng==
- nhưng khi thiết kế mới cho hệ thống sống lâu, nên theo dõi `Gateway API` và lộ trình migration của ingress controller

## Checklist debug thực chiến khi cert không ra

Thứ tự kiểm tra nên là:

1. `kubectl get ingress`
2. `kubectl describe ingress <name>`
3. `kubectl get certificate`
4. `kubectl describe certificate <name>`
5. `kubectl get certificaterequest`
6. `kubectl get order`
7. `kubectl get challenge`
8. `kubectl describe challenge <name>`
9. `kubectl get secret <tls-secret>`

Nếu là `HTTP01`, hãy kiểm tra thêm:

- domain có trỏ đúng tới ingress/load balancer chưa
- port `80` có reachable từ Internet không
- challenge path có bị app rewrite hoặc redirect sai không
- ingress class của app và solver có đúng không

Nếu là `DNS01`, hãy kiểm tra thêm:

- TXT record `_acme-challenge` đã được tạo đúng chưa
- DNS đã propagate chưa
- credential/API quyền cập nhật DNS có đủ không

## Cách chọn nhanh trong thực tế

| Tình huống | Nên chọn |
|---|---|
| website/app public, 1 vài host thường | `ACME + HTTP01` |
| website/app public, cần wildcard | `ACME + DNS01` |
| internal PKI / mTLS nội bộ | `CA` hoặc `Vault` |
| lab hoặc bootstrap root CA | `SelfSigned` rồi nâng lên `CA` |
| enterprise có platform certificate management riêng | `CyberArk` hoặc external issuer phù hợp |

## Ghi nhớ ngắn nhất

==`Ingress` nói traffic phải đi đâu.==

==`ingress controller` là người thật nhận traffic và phục vụ HTTPS.==

==`cert-manager` lo certificate phải được cấp và gia hạn thế nào.==

==`Secret` là điểm giao nhau giữa cert-manager và ingress controller.==

==`Issuer`/`ClusterIssuer` quyết định nguồn ký.==

==`HTTP01` là đường dễ nhất cho host public thường; `DNS01` là đường phải nghĩ tới ngay khi nghe chữ wildcard.==
