---
doc_id: longhorn-network-iscsi-node3-debug-runbook
title: Runbook debug Longhorn -> Network -> iSCSI khi node3 khÃ´ng báº¯t Ä‘Æ°á»£c Longhorn
legacy_title: Runbook debug Longhorn - Network - iSCSI node3
type: runbook
domain: storage
keywords:
  - longhorn
  - network
  - iscsi
  - node3
  - csi
  - debug
status: active
---

# Runbook debug Longhorn -> Network -> iSCSI khi node3 không bắt được Longhorn

> Mục tiêu của tài liệu này là biến một case debug thực tế thành **một quy trình suy luận có thứ tự**.
>
> Điểm quan trọng nhất không phải là nhớ thật nhiều lệnh, mà là hiểu:
> - đang kiểm tra ở tầng nào
> - vì sao phải kiểm tra tầng đó trước
> - output nào là tín hiệu quyết định
> - khi nào được phép kết luận và chuyển sang tầng tiếp theo

---

## 1. Kết luận ngắn gọn của toàn bộ case

Case này **không phải lỗi Longhorn đơn thuần**.

Chuỗi root cause đúng là:

1. `node3` join cluster **sai version line** so với `node1/node2`
2. `node3` có vấn đề ở **pod network / overlay cross-node**
3. `Longhorn CSI` fail chỉ là **hậu quả** vì không reach được `longhorn-backend`
4. `DNS` fail cũng chỉ là **triệu chứng**, chưa phải root cause đầu tiên
5. Sau khi network hồi, `Longhorn` vẫn còn **stale disk metadata**
6. Một `volume` thật của `Postgres` cùng `replica cũ` đang chặn cleanup `node3`
7. `iSCSI` là tầng kiểm tra tiếp theo nếu network đã ổn, Longhorn đã attach đúng, nhưng volume vẫn không mount/không publish vào node

Nếu chỉ nhìn UI hoặc chỉ nhìn `longhorn-csi-plugin`, rất dễ chẩn đoán sai thành:

- "Longhorn bị lỗi"
- "CoreDNS bị lỗi"
- "registrar bị lỗi"

Trong khi thực tế phải chứng minh theo thứ tự từ **cluster -> network -> Longhorn control plane -> Longhorn metadata -> volume attachment -> host storage path**.

---

## 2. Tư duy đúng để debug case kiểu này

## Nguyên tắc vàng

Khi một pod storage như Longhorn bị lỗi, không được kết luận ngay là storage hỏng.

Phải đi theo thứ tự:

1. **Node có gia nhập cluster đúng không**
2. **Longhorn fail ở tầng control plane, data plane, hay host path**
3. **Pod trên node đó có reach được service/pod IP cross-node hay không**
4. **DNS fail là nguyên nhân hay chỉ là hậu quả của network**
5. **Sau khi network ổn, Longhorn còn giữ metadata cũ hay không**
6. **Có volume/engine/replica nào đang giữ node đó không**
7. **Nếu Longhorn attach đúng nhưng node vẫn mount fail thì mới xuống iSCSI**

## Vì sao phải check tuần tự như vậy

Vì mỗi tầng phía dưới là điều kiện tiên quyết cho tầng phía trên:

- Node join sai hoặc state lệch -> CNI/overlay có thể lệch
- Overlay lệch -> pod không reach được service/backend
- Không reach backend -> `longhorn-csi-plugin` không lên
- `longhorn-csi-plugin` không lên -> `node-driver-registrar` chỉ đứng chờ socket
- Network sửa xong nhưng Longhorn vẫn giữ state cũ -> UI vẫn xấu, disk vẫn `0`
- Metadata sửa xong nhưng volume thật còn attached -> không xóa sạch node record được
- Tất cả đều đúng mà mount vẫn fail -> lúc đó mới hợp lý để soi iSCSI

Nói ngắn gọn:

> Không debug từ triệu chứng gần mắt nhất.  
> Debug từ tầng nền nhất có khả năng làm hỏng toàn bộ các tầng phía trên.

---

## 3. Sơ đồ suy luận của toàn bộ case

```text
node3 không bắt được Longhorn
        |
        v
Longhorn CSI trên node3 lỗi
        |
        v
Cần hỏi: CSI lỗi do chính nó, hay do không reach được backend?
        |
        v
Test service/DNS/pod IP từ chính pod trên node3
        |
        +--> Nếu DNS fail nhưng pod IP direct vẫn được -> nghi CoreDNS/service discovery
        |
        +--> Nếu pod IP cross-node cũng timeout -> root cause ở overlay/pod network
        |
        v
Khi network ổn lại
        |
        v
Kiểm tra Longhorn node CRD/disk metadata
        |
        +--> Nếu diskUUID mismatch / capacity = 0 -> stale metadata
        |
        v
Muốn cleanup node Longhorn
        |
        v
Kiểm tra engine / replica / volumeattachment đang bám node nào
        |
        v
Map volume Longhorn -> PV -> PVC -> workload thật
        |
        v
Detach workload, xóa replica stale, rồi cleanup node
        |
        v
Nếu attach đúng mà node vẫn không mount được volume
        |
        v
Kiểm tra tầng iSCSI trên host
```

---

## 4. Giai đoạn 1 - Xác nhận node3 có vào cluster đúng chưa

## Mục tiêu

Trước khi đụng vào Longhorn, phải xác định `node3` có phải là một node "khỏe và đồng bộ" với cluster hay không.

## Lệnh

```bash
kubectl get nodes -o wide
kubectl describe node node3
```

## Kết quả đã đọc được

- `node1`, `node2`: `v1.33.5+rke2r1`
- `node3`: `v1.34.6+rke2r1`
- `node3` vẫn `Ready`
- không có taint lạ
- `NetworkUnavailable=False`

## Kết luận

`node3` **có join vào cluster**, nhưng join **khác version line** với cluster đang chạy.

## Ý nghĩa của bước này

Đây là bước để trả lời câu hỏi:

> "Node có thật sự là một member đúng chuẩn của cluster không, hay chỉ là nhìn bề ngoài vẫn Ready?"

Một node vẫn có thể `Ready` nhưng vẫn sai version line, sai role history, hoặc mang state cũ từ lần cài đặt trước.

## Vì sao phải check bước này đầu tiên

Vì nếu ngay từ nền node đã lệch, thì mọi lỗi phía trên như:

- Longhorn không lên
- DNS timeout
- pod network bất thường

đều có thể là hậu quả dây chuyền.

Nếu bỏ qua bước này, ta rất dễ lao vào sửa Longhorn trong khi cluster membership đã lệch từ đầu.

## Hành động đúng sau bước này

Ghi nhận `node3` là node có rủi ro cao:

- join sai version line
- không được xem là node "sạch"
- mọi chẩn đoán sau đó phải nhớ rằng lỗi có thể nằm ở tầng nền của node

---

## 5. Giai đoạn 2 - Xác nhận Longhorn chết ở tầng nào

## Mục tiêu

Không được kết luận "Longhorn lỗi" một cách mơ hồ. Phải xác định Longhorn đang fail ở đâu:

- plugin không start được?
- registrar lỗi?
- hay plugin không reach được backend?

## Lệnh

```bash
kubectl get pods -n longhorn-system -o wide
kubectl describe pod longhorn-csi-plugin-5t9mg -n longhorn-system
kubectl logs longhorn-csi-plugin-5t9mg -n longhorn-system -c longhorn-csi-plugin --tail=100
kubectl logs longhorn-csi-plugin-5t9mg -n longhorn-system -c node-driver-registrar --tail=100
```

## Kết quả đã đọc được

- `longhorn-csi-plugin` trên `node3` không healthy
- `node-driver-registrar` đứng chờ `/csi/csi.sock`
- `longhorn-csi-plugin` báo:
  - `Failed to initialize Longhorn API client`
  - `Get "http://longhorn-backend:9500/v1": context deadline exceeded`
- `registrar` báo:
  - vẫn đang connect tới `/csi/csi.sock`
  - rồi timeout

## Kết luận

Không được sửa `registrar` trước.

`registrar` chỉ là **nạn nhân** vì `plugin` chính chưa lên.

Kết luận đúng phải là:

> `longhorn-csi-plugin` không nói chuyện được với `longhorn-backend`

## Ý nghĩa của bước này

Bước này dùng để tách:

- **triệu chứng phụ**: registrar chờ socket
- **triệu chứng chính**: plugin không init được API client
- **hướng debug đúng**: connectivity từ pod tới backend

## Vì sao phải check bước này trước khi đi kiểm tra DNS/network

Vì nếu chưa đọc log đúng container, ta chưa biết mình cần debug theo hướng nào.

Sau bước này ta mới có cơ sở để đi tiếp theo hướng:

- service reachability
- DNS
- pod network

chứ không phải chỉnh mù vào CSI sidecar.

---

## 6. Giai đoạn 3 - Chứng minh lỗi nằm ở pod network / DNS chứ không riêng Longhorn

## Mục tiêu

Đây là bước bản lề nhất của toàn bộ case.

Phải chứng minh:

- lỗi chỉ nằm ở DNS?
- hay lỗi sâu hơn là pod network cross-node?

## Lệnh

```bash
kubectl exec -it nettest-node3 -- sh

nslookup kubernetes.default.svc
nslookup longhorn-backend.longhorn-system.svc

wget -T 3 -S -O- http://longhorn-backend.longhorn-system.svc:9500/v1
wget -T 3 -S -O- http://10.42.0.37:9500/v1
wget -T 3 -S -O- http://10.42.1.4:9500/v1
wget -T 3 -S -O- http://10.42.3.5:9500/v1
```

## Kết quả đã đọc được

- `nslookup kubernetes.default.svc` timeout
- `nslookup longhorn-backend.longhorn-system.svc` timeout
- gọi FQDN thì `bad address`
- gọi trực tiếp pod IP `10.42.0.37` và `10.42.1.4` thì timeout
- gọi `10.42.3.5` thì `connection refused`

## Kết luận

Đây không còn là lỗi DNS đơn thuần.

Lý do:

- Nếu chỉ DNS lỗi, thì gọi **pod IP trực tiếp** vẫn phải đi được
- Nhưng ở đây pod IP cross-node cũng timeout

=> Kết luận đúng là:

> `pod network / overlay giữa node3 và node khác đang hỏng`

`DNS fail` chỉ là triệu chứng kéo theo.

## Ý nghĩa của từng lệnh trong bước này

`nslookup kubernetes.default.svc`

- kiểm tra DNS nền của cluster có trả lời không
- nếu lệnh này fail, nghĩa là pod trên `node3` không phân giải nổi service cơ bản

`nslookup longhorn-backend.longhorn-system.svc`

- kiểm tra riêng service Longhorn backend
- giúp biết lỗi có phải chỉ riêng Longhorn service name hay không

`wget` tới FQDN service

- kiểm tra đường đi đầy đủ: DNS -> service -> backend

`wget` tới pod IP trực tiếp

- cắt DNS ra khỏi bài toán
- chỉ còn bài toán reachability ở tầng overlay/pod network

## Vì sao phải check lần lượt theo thứ tự này

Thứ tự này giúp loại trừ rất nhanh:

1. test bằng DNS name
2. nếu fail, test thẳng pod IP
3. nếu pod IP vẫn fail, kết luận được ngay network cross-node có vấn đề

Nếu làm ngược, ta sẽ khó phân biệt:

- lỗi DNS
- lỗi service
- lỗi endpoint
- lỗi overlay

Đây chính là bước giúp tránh chẩn đoán sai rằng "Longhorn backend chết".

---

## 7. Giai đoạn 4 - Kiểm tra DNS service để tránh đoán mò

## Mục tiêu

Sau khi nghi network/DNS, cần kiểm tra `CoreDNS` và addon layer bằng dữ kiện thật, không đoán cảm tính.

## Lệnh

```bash
kubectl get svc -n kube-system rke2-coredns-rke2-coredns
kubectl get endpoints -n kube-system rke2-coredns-rke2-coredns
kubectl get pods -n kube-system -o wide | grep coredns

kubectl describe pod -n kube-system rke2-coredns-rke2-coredns-6464f98784-5qb44
kubectl describe pod -n kube-system rke2-coredns-rke2-coredns-autoscaler-67bb49dff-wxtwb
kubectl describe pod -n kube-system rke2-metrics-server-75d485c65b-x4tdl
kubectl describe pod -n kube-system rke2-snapshot-controller-696989ffdd-2gf64
kubectl describe pod -n kube-system rke2-canal-qx89q
```

## Kết quả đã đọc được

- DNS service là `10.43.0.10`
- endpoint thực tế chỉ có `10.42.0.68` trên `node1`
- replica `CoreDNS` ở `node2` là `Pending`
- `autoscaler`, `metrics-server`, `snapshot-controller`, `canal` trên `node2` cũng `Pending`
- nhiều pod `Pending` có `PodScheduled=True` nhưng không có IP, không có event hữu ích

## Kết luận

Không chỉ `node3` có vấn đề.

`addon/network layer` của cluster cũng đang bất ổn, đặc biệt ở `node2`.

## Ý nghĩa của bước này

Bước này rất quan trọng vì nó ngăn ta kết luận quá hẹp rằng:

> "Longhorn hỏng riêng trên node3"

Trong khi dữ liệu thật cho thấy:

- cluster addon layer cũng không khỏe
- network/CNI không phải chỉ ảnh hưởng một pod Longhorn

## Vì sao phải check bước này sau bước 3

Vì bước 3 đã chứng minh được:

- đây không còn là lỗi riêng của Longhorn
- network/DNS của cluster đáng nghi

Lúc đó mới hợp lý để đi kiểm tra `CoreDNS`, `canal`, `metrics-server`, `snapshot-controller`.

Nếu check addon quá sớm, ta sẽ có rất nhiều tín hiệu nhiễu mà chưa biết nó liên quan trực tiếp hay không.

---

## 8. Giai đoạn 5 - Kiểm tra kube-proxy / canal trên node3 có chết hẳn không

## Mục tiêu

Phân biệt 2 kiểu lỗi:

- lỗi thô: CNI hoặc kube-proxy chết hẳn, log đỏ rõ ràng
- lỗi khó: process vẫn chạy, nhưng network state lệch hoặc stale

## Lệnh

```bash
kubectl logs -n kube-system kube-proxy-node3 --tail=200
kubectl logs -n kube-system rke2-canal-8gdd6 --tail=200
```

## Kết quả đã đọc được

- `kube-proxy` sync cache bình thường
- `canal/felix` vẫn thấy workload endpoint up
- không có lỗi "chết hẳn" quá rõ ở mức pod log

## Kết luận

Đây không giống kiểu:

> CNI chết sạch hoặc kube-proxy crash rõ ràng

Nó giống kiểu:

> `node3` join được, process vẫn sống, nhưng overlay/network state giữa node3 và cluster bị lệch

## Ý nghĩa của bước này

Bước này giải thích vì sao về sau hướng xử lý đúng là:

- không vá từng pod
- không ngồi restart sidecar lẻ tẻ
- mà chuyển sang hướng **làm sạch node3 và join lại**

## Vì sao phải check bước này

Vì nếu `canal` hay `kube-proxy` crash thật thì cách xử lý sẽ khác hẳn.

Nhưng khi log không cho thấy hỏng hoàn toàn, trong khi test thực tế vẫn fail cross-node, ta hiểu đây là case **state lệch / join bẩn / overlay inconsistency**.

---

## 9. Giai đoạn 6 - Quyết định chiến lược đúng: làm sạch và join lại node3

## Vì sao quyết định này đúng

Tại thời điểm này đã có đủ bằng chứng:

- `node3` join sai version line
- `node3` từng là master rồi đổi sang worker
- pod network cross-node lỗi
- Longhorn CSI chỉ fail theo
- `node3` khác OS so với `node1/node2`

## Cách hiểu đúng

Với case kiểu:

- đổi role node
- đổi OS
- có dấu hiệu CNI lệch
- storage fail theo

thì tiếp tục vá từng lỗi nhỏ rất dễ sa lầy.

Cách đúng là:

> `drain + gỡ sạch + join lại node3 như một worker mới`

## Ý nghĩa của bước này

Đây không phải "bỏ cuộc debug", mà là một quyết định kiến trúc đúng.

Khi state nền của node không còn đáng tin, cleanup và join lại thường nhanh hơn, sạch hơn, và đáng tin hơn so với:

- restart từng daemon
- sửa từng pod
- vá từng manifest

---

## 10. Giai đoạn 7 - Xác nhận node3 sau khi join lại

## Mục tiêu

Sau khi rejoin, phải kiểm tra xem node có nhận được network state mới hay chưa.

## Lệnh

```bash
kubectl get pod -n kube-system
kubectl get pods -n longhorn-system -o wide
```

## Kết quả đã đọc được

- `node3` bắt đầu có pod IP mới dạng `10.42.3.x`
- `rke2-canal` trên `node3` lên
- `kube-proxy-node3` lên
- `rke2-ingress-nginx-controller` lên trên `node3`
- `longhorn-manager` và `longhorn-csi-plugin` trên `node3` quay lại chạy

## Kết luận

`node3` đã rejoin sạch hơn trước.

Việc pod IP đổi từ line cũ sang `10.42.3.x` là tín hiệu mạnh cho thấy:

- `PodCIDR/state` đã được cấp phát lại
- overlay state đã đổi

## Ý nghĩa của bước này

Đây là bước xác nhận rằng hành động "làm sạch và join lại" thực sự có tác dụng ở tầng network/control plane, chứ không chỉ làm đẹp trạng thái pod.

---

## 11. Giai đoạn 8 - Bước turning point thật sự: restart Canal ở node1

## Mục tiêu

Kiểm tra xem đường pod network cross-node giữa `node3` và `node1` có thực sự thông lại chưa.

## Lệnh

```bash
kubectl delete pod -n kube-system rke2-canal-wbqnp
kubectl get pod -n kube-system

kubectl exec -it nettest-node3 -- sh
nslookup kubernetes.default.svc
wget -T 3 -S -O- http://10.42.0.37:9500/v1
```

## Kết quả đã đọc được

Sau khi restart `canal` trên `node1`:

- gọi `http://10.42.0.37:9500/v1` từ `nettest-node3` trả `HTTP/1.1 200 OK`
- `nslookup` không còn timeout, mà chuyển thành `NXDOMAIN`

## Kết luận

Cross-node pod network giữa `node3` và `node1` đã thông lại.

Đây là turning point của cả case.

## Vì sao `NXDOMAIN` ở đây lại còn tốt hơn timeout

`timeout` nghĩa là pod không reach được DNS path hoặc DNS server không trả lời.

`NXDOMAIN` nghĩa là:

- request đã đi được tới DNS
- DNS đã trả lời
- chỉ là tên đang hỏi không tồn tại hoặc không đúng

Tức là về mặt network, hệ thống đã **tiến lên một tầng tốt hơn hẳn**.

## Ý nghĩa của bước này

Nó chứng minh trực tiếp bằng dữ kiện:

- trước đó pod IP cross-node timeout
- sau đó pod IP cross-node trả `200 OK`

Không còn là suy đoán nữa, mà là bằng chứng rằng **overlay path đã hồi**.

---

## 12. Giai đoạn 9 - Xác nhận Longhorn CSI không còn là vấn đề chính

## Mục tiêu

Sau khi network tốt lên, kiểm tra CSI có hồi theo hay không để xác nhận giả thuyết ban đầu.

## Lệnh

```bash
kubectl describe pod longhorn-csi-plugin-qwp99 -n longhorn-system
kubectl logs longhorn-csi-plugin-qwp99 -n longhorn-system -c longhorn-csi-plugin --tail=100
kubectl logs longhorn-csi-plugin-qwp99 -n longhorn-system -c node-driver-registrar --tail=100
```

## Kết quả đã đọc được

- `longhorn-csi-plugin` trên `node3` chạy lại với IP mới `10.42.3.4`
- đôi lúc vẫn còn log:
  - `lookup longhorn-backend on 10.43.0.10:53: i/o timeout`
- `registrar` tiếp tục chờ socket khi plugin chưa lên xong

## Kết luận

CSI vẫn chỉ là hậu quả.

Mỗi lần network tốt lên, CSI cũng tốt lên theo.

Điều này xác nhận hướng debug từ đầu là đúng.

## Ý nghĩa của bước này

Bước này là bước "đóng vòng lặp suy luận".

Ban đầu ta giả thuyết:

> CSI fail vì không reach được backend

Sau khi network hồi, CSI hồi theo.

=> giả thuyết được kiểm chứng.

---

## 13. Giai đoạn 10 - Tìm ra root cause cuối ở Longhorn: stale disk metadata

## Mục tiêu

Khi network đã hồi mà UI Longhorn/node state vẫn kỳ lạ, phải kiểm tra `Longhorn Node CRD`.

## Lệnh

```bash
kubectl get nodes.longhorn.io node3 -n longhorn-system -o yaml
```

## Kết quả đã đọc được

Trong `diskStatus` có:

- `DiskFilesystemChanged`
- `record diskUUID doesn't match the one on the disk`
- `storageAvailable: 0`
- `storageMaximum: 0`

## Kết luận

Đây là root cause của phần Longhorn UI/node state ở giai đoạn sau:

> `node3` đã sống lại, nhưng disk Longhorn trên node3 vẫn là record cũ

Tức là:

- node đã rebuild/rejoin
- nhưng Longhorn còn giữ metadata disk cũ

## Ý nghĩa của bước này

Bước này giúp tách rất rõ 2 lớp vấn đề:

1. **network/control plane** đã tốt hơn
2. **Longhorn metadata** vẫn stale

Nếu không kiểm tra CRD này, ta sẽ tưởng network vẫn chưa fix xong.

Trong khi thật ra network có thể đã ổn, nhưng Longhorn đang bị state cũ giữ lại.

---

## 14. Giai đoạn 11 - Cố xóa Longhorn node record và đọc đúng blocker

## Mục tiêu

Không xóa mù. Phải đọc chính xác Longhorn đang chặn vì điều gì.

## Lệnh

```bash
kubectl delete nodes.longhorn.io node3 -n longhorn-system
```

## Kết quả đã đọc được

Longhorn lần lượt chặn vì:

- còn `1 replica`, `1 engine running on it`
- sau đó còn `0 replica`, `1 engine running on it`
- cuối cùng còn vì node vẫn `Ready` dù đã `schedulable false`

## Kết luận

Thông báo lỗi của Longhorn chính là một checklist blocker rất giá trị:

- còn replica không?
- còn engine không?
- node còn ready không?

## Ý nghĩa của bước này

Đây là bài học quan trọng:

> Error message của Longhorn không chỉ là báo lỗi, mà là danh sách điều kiện cleanup chưa đạt.

Nếu đọc kỹ, ta không cần đoán mò nữa.

---

## 15. Giai đoạn 12 - Dùng Longhorn CRD để tìm chính xác volume nào đang chặn node3

## Mục tiêu

Không đoán theo UI. Tra thẳng CRD để biết volume nào đang giữ node.

## Lệnh

```bash
kubectl get engines.longhorn.io -n longhorn-system -o yaml | grep -n -C3 "node3"
kubectl get replicas.longhorn.io -n longhorn-system -o yaml | grep -n -C3 "node3"
kubectl get volumeattachments.longhorn.io -n longhorn-system -o yaml | grep -n -C3 "node3"
```

## Kết quả đã đọc được

- `engine` chạy trên `node3` là của volume `pvc-b45aa67b-8948-4901-8d8b-44ecc4afbd73`
- còn một `replica` cũ bám `node3`
- `volumeattachment` cũng trỏ về volume đó

## Kết luận

Đây là bước rất quan trọng vì nó trả lời đúng câu hỏi:

> volume nào đang thật sự giữ node3 lại?

## Ý nghĩa của từng CRD

`engines.longhorn.io`

- cho biết volume engine đang chạy ở đâu
- rất hữu ích để biết volume còn attached vào node nào

`replicas.longhorn.io`

- cho biết replica dữ liệu nào còn tồn tại trên node
- giúp phát hiện replica stale sau khi rebuild node

`volumeattachments.longhorn.io`

- cho biết yêu cầu attach/detach hiện tại của volume
- giúp xác nhận volume đã được giải phóng khỏi node hay chưa

## Vì sao bước này phải đứng trước khi map sang PVC

Vì trước tiên phải xác định **tên volume Longhorn cụ thể**.

Chỉ khi biết volume nào chặn node, ta mới map ngược sang:

- PV
- PVC
- Pod/StatefulSet thật

---

## 16. Giai đoạn 13 - Map Longhorn volume về PVC thật trong Kubernetes

## Mục tiêu

Nối tầng Longhorn với workload Kubernetes thật đang dùng volume.

## Lệnh

```bash
kubectl get pv pvc-b45aa67b-8948-4901-8d8b-44ecc4afbd73 -o jsonpath='{.spec.claimRef.namespace}{" "}{.spec.claimRef.name}{"\n"}'
kubectl describe pvc -n database data-postgres-sts-0
```

## Kết quả đã đọc được

Volume đó map về PVC:

- namespace: `database`
- PVC: `data-postgres-sts-0`

PVC này đang được dùng bởi:

- `postgres-sts-0`

## Kết luận

Blocker thật lúc này không còn là:

> "Longhorn không chịu xóa node3"

Mà là:

> `Postgres volume` đang attached vào `node3`

## Ý nghĩa của bước này

Đây là điểm nối giữa:

- tầng Longhorn storage
- tầng Kubernetes workload

Nếu không map đến tận PVC và pod thật, ta sẽ chỉ thấy object storage mơ hồ mà không biết phải detach workload nào.

---

## 17. Giai đoạn 14 - Detach workload để giải phóng engine

## Mục tiêu

Giải phóng volume khỏi node3 bằng cách dừng workload thật đang sử dụng nó.

## Lệnh

```bash
kubectl cordon node3
kubectl scale sts -n database postgres-sts --replicas=0
kubectl get engines.longhorn.io -n longhorn-system -o yaml | grep -n -C3 "pvc-b45aa67b-8948-4901-8d8b-44ecc4afbd73"
kubectl get volumeattachments.longhorn.io -n longhorn-system -o yaml | grep -n -C3 "pvc-b45aa67b-8948-4901-8d8b-44ecc4afbd73"
```

## Kết quả đã đọc được

Sau khi scale xuống:

- `attachmentTickets: {}`
- `nodeID: ""`
- engine không còn attached vào `node3`

## Kết luận

Đây là bước gỡ được blocker:

> `1 engine running on it`

## Ý nghĩa của bước này

Longhorn không thể cleanup node nếu volume còn đang dùng thật.

Cho nên phải dừng workload trước, rồi mới đòi storage detach sạch.

---

## 18. Giai đoạn 15 - Xóa replica stale

## Mục tiêu

Xóa replica cũ còn sót lại sau quá trình rebuild/rejoin node.

## Lệnh

```bash
kubectl delete replicas.longhorn.io pvc-0c9e60ba-c8f2-4c2a-9e8e-b177bd16462c-r-02c8bc33 -n longhorn-system
```

## Kết quả đã đọc được

Replica stale đã bị xóa.

Sau đó blocker giảm dần:

- từ `1 replica, 1 engine`
- thành `0 replica, 1 engine`
- rồi tiếp tục giảm về sau

## Kết luận

Đây là bước gỡ blocker:

> `replica stale`

## Ý nghĩa của bước này

Node mới nhưng Longhorn vẫn còn object replica cũ là tình huống rất hay gặp sau rebuild node hoặc đổi disk identity.

Nếu không xóa phần stale này, UI và cleanup logic của Longhorn sẽ tiếp tục bị kẹt.

---

## 19. Giai đoạn 16 - Patch scheduling để hiểu đúng vì sao UI vẫn khác

## Mục tiêu

Tách bạch:

- Kubernetes cordon
- Longhorn scheduling

## Lệnh

```bash
kubectl patch nodes.longhorn.io node3 -n longhorn-system --type merge -p '
spec:
  allowScheduling: false
  disks:
    default-disk-fd0000000000:
      allowScheduling: false
'

kubectl get nodes.longhorn.io node3 -n longhorn-system -o yaml
```

## Kết quả đã đọc được

- `spec.allowScheduling: false`
- disk `allowScheduling: false`
- condition `Schedulable=False`
- reason: `KubernetesNodeCordoned`

## Kết luận

Lệnh này giúp giải thích rõ vì sao UI `node3` hiện `Disabled`, không giống `node1/node2`.

## Ý nghĩa của bước này

Đây là bước rất hay để tránh nhầm:

- Kubernetes thấy node còn `Ready`
- nhưng Longhorn có thể đang không cho scheduling

Hai khái niệm này liên quan nhau nhưng không hoàn toàn giống nhau.

---

## 20. Phần iSCSI - Khi nào phải debug tiếp xuống tầng iSCSI

## Khi nào mới nên kiểm tra iSCSI

Chỉ kiểm tra `iSCSI` khi các điều kiện sau đã đúng:

1. pod network cross-node đã thông
2. DNS/service không còn là blocker chính
3. `longhorn-csi-plugin` đã lên hoặc ít nhất backend đã reach được
4. Longhorn volume đã attach đúng về node
5. nhưng workload vẫn mount fail, publish fail, hoặc attach xong mà device không xuất hiện đúng trên node

Nói cách khác:

> iSCSI là tầng thấp hơn của khâu attach/mount trên host.  
> Không được nhảy xuống iSCSI khi network và Longhorn control plane còn đang hỏng.

## Vì sao không nên check iSCSI quá sớm

Vì nếu:

- backend còn timeout
- DNS còn fail
- pod IP cross-node còn không reach được

thì mọi lỗi iSCSI nhìn thấy lúc đó chỉ là hậu quả hoặc nhiễu.

Lúc ấy sửa iSCSI sẽ không giải quyết được gốc vấn đề.

## Dấu hiệu cho thấy nên kiểm tra iSCSI

- Longhorn UI/CRD cho thấy volume đã attached vào đúng node
- `volumeattachment` đã có node đích rõ ràng
- workload vẫn báo mount error hoặc device path không xuất hiện
- host path tới volume không hiện đúng dù control plane đã ổn

## Các bước kiểm tra iSCSI nên làm

> Phần này là checklist thực chiến để dùng sau khi tầng network và Longhorn đã ổn.

### 20.1 Kiểm tra service iSCSI trên node

Ví dụ tùy OS:

```bash
systemctl status iscsid
systemctl status iscsi
```

## Ý nghĩa

- xác nhận daemon iSCSI có chạy không
- nếu daemon chết thì host không thể login session tới target

### 20.2 Kiểm tra kernel module liên quan

```bash
lsmod | grep iscsi
lsmod | grep scsi
```

## Ý nghĩa

- xác nhận các module cần thiết đã được nạp
- thiếu module thì attach có thể fail dù control plane báo bình thường

### 20.3 Kiểm tra session iSCSI hiện có

```bash
iscsiadm -m session
```

## Ý nghĩa

- xem node đã login vào target nào chưa
- nếu volume Longhorn đã attach mà không có session tương ứng thì vấn đề đang ở tầng host/iSCSI

### 20.4 Kiểm tra log hệ thống liên quan iSCSI

```bash
journalctl -u iscsid --since "1 hour ago"
journalctl -u iscsi --since "1 hour ago"
```

## Ý nghĩa

- tìm lỗi login timeout
- authentication error
- reconnect loop
- target discovery fail

### 20.5 Kiểm tra device block sau khi attach

```bash
lsblk
blkid
```

## Ý nghĩa

- xác nhận volume sau khi attach có hiện thành block device không
- nếu Longhorn nói attached nhưng host không có device mới thì phải xem lại iSCSI path

## Tóm tắt vai trò của iSCSI trong chuỗi debug

`Longhorn` cần nhiều tầng cùng đúng:

1. Kubernetes node state đúng
2. Pod network đúng
3. Service/backend reach được
4. CSI plugin hoạt động đúng
5. Longhorn metadata không stale
6. Volume attach đúng
7. Host path/iSCSI mount thành công

`iSCSI` chỉ là **bước cuối hơn** trong chuỗi này, không phải nơi bắt đầu.

---

## 21. Tổng kết root cause theo đúng thứ tự command đã chứng minh

## Root cause 1

`node3` join sai version line với cluster.

Lệnh phát hiện:

```bash
kubectl get nodes -o wide
kubectl describe node node3
```

## Root cause 2

Pod network / overlay giữa `node3` và cluster bị hỏng.

Lệnh phát hiện:

```bash
kubectl exec -it nettest-node3 -- sh
nslookup kubernetes.default.svc
wget -T 3 -S -O- http://10.42.0.37:9500/v1
```

## Root cause 3

Longhorn CSI fail chỉ là hậu quả vì không reach được `longhorn-backend`.

Lệnh phát hiện:

```bash
kubectl describe pod longhorn-csi-plugin-<pod> -n longhorn-system
kubectl logs longhorn-csi-plugin-<pod> -n longhorn-system -c longhorn-csi-plugin --tail=100
kubectl logs longhorn-csi-plugin-<pod> -n longhorn-system -c node-driver-registrar --tail=100
```

## Root cause 4

Cluster addon layer cũng đang rối, nhất là `node2` có nhiều pod `Pending`.

Lệnh phát hiện:

```bash
kubectl get svc -n kube-system rke2-coredns-rke2-coredns
kubectl get endpoints -n kube-system rke2-coredns-rke2-coredns
kubectl get pods -n kube-system -o wide | grep coredns
kubectl describe pod -n kube-system ...
```

## Root cause 5

Sau khi network hồi, disk metadata của Longhorn trên `node3` vẫn stale.

Lệnh phát hiện:

```bash
kubectl get nodes.longhorn.io node3 -n longhorn-system -o yaml
```

## Root cause 6

Một volume Postgres và replica cũ chặn việc cleanup `node3`.

Lệnh phát hiện:

```bash
kubectl get engines.longhorn.io -n longhorn-system -o yaml | grep -n -C3 "node3"
kubectl get replicas.longhorn.io -n longhorn-system -o yaml | grep -n -C3 "node3"
kubectl get volumeattachments.longhorn.io -n longhorn-system -o yaml | grep -n -C3 "node3"
kubectl get pv <volume-name> -o jsonpath='{.spec.claimRef.namespace}{" "}{.spec.claimRef.name}{"\n"}'
kubectl describe pvc -n <ns> <pvc-name>
```

## Root cause 7

`iSCSI` chỉ là tầng cần kiểm tra nếu mọi thứ phía trên đã đúng mà volume vẫn không mount được vào node/workload.

---

## 22. Bộ lệnh quan trọng nhất nếu phải debug lại từ đầu

Nếu phải rút gọn thành bộ lệnh đáng giá nhất, nên đi đúng thứ tự này:

```bash
kubectl get nodes -o wide
kubectl describe node node3

kubectl get pods -n longhorn-system -o wide
kubectl describe pod longhorn-csi-plugin-<pod> -n longhorn-system
kubectl logs longhorn-csi-plugin-<pod> -n longhorn-system -c longhorn-csi-plugin --tail=100
kubectl logs longhorn-csi-plugin-<pod> -n longhorn-system -c node-driver-registrar --tail=100

kubectl exec -it nettest-node3 -- sh
nslookup kubernetes.default.svc
wget -T 3 -S -O- http://10.42.0.37:9500/v1

kubectl get svc -n kube-system rke2-coredns-rke2-coredns
kubectl get endpoints -n kube-system rke2-coredns-rke2-coredns
kubectl get pods -n kube-system -o wide | grep coredns
kubectl describe pod -n kube-system rke2-canal-<pod>

kubectl get nodes.longhorn.io node3 -n longhorn-system -o yaml

kubectl get engines.longhorn.io -n longhorn-system -o yaml | grep -n -C3 "node3"
kubectl get replicas.longhorn.io -n longhorn-system -o yaml | grep -n -C3 "node3"
kubectl get volumeattachments.longhorn.io -n longhorn-system -o yaml | grep -n -C3 "node3"

kubectl get pv <volume-name> -o jsonpath='{.spec.claimRef.namespace}{" "}{.spec.claimRef.name}{"\n"}'
kubectl describe pvc -n <ns> <pvc-name>
```

Nếu đến đây mọi thứ đều đúng mà volume vẫn attach/mount fail, mới đi tiếp:

```bash
systemctl status iscsid
iscsiadm -m session
lsblk
journalctl -u iscsid --since "1 hour ago"
```

---

## 23. Câu chốt cần nhớ

Cái khó của case này không phải là "biết Longhorn lỗi", mà là dùng đúng lệnh để **lần lượt chứng minh**:

- `node3` join sai version
- pod network cross-node hỏng
- CSI chỉ là hậu quả
- DNS fail chỉ là triệu chứng
- disk metadata Longhorn bị stale
- volume Postgres thật đang chặn cleanup
- và chỉ khi các tầng trên đã đúng mới có lý do để kiểm tra tiếp `iSCSI`

Đó mới là chuỗi suy luận đúng giúp fix xong case `node3 không bắt được Longhorn`.

---

## 24. Bản ghi nhớ siêu ngắn

Khi gặp lại case tương tự, hãy tự hỏi đúng 5 câu này:

1. `node` có join đúng version line và state sạch chưa?
2. `Longhorn CSI` fail vì chính nó, hay vì không reach được backend?
3. Pod trên node lỗi có reach được pod IP cross-node không?
4. Sau khi network hồi, Longhorn có còn giữ stale metadata không?
5. Nếu volume đã attach đúng mà vẫn không mount được, iSCSI trên host có khỏe không?

Nếu trả lời đúng 5 câu đó theo đúng thứ tự, gần như sẽ không đi sai hướng debug.


==== HISTORY COMMAND TO CHECK IN THE REAL ENV:

829  kubectl get nodes.longhorn.io -n longhorn-system                                                                                                                                                                                                                     

  830  kubectl get pods -n longhorn-system -o wide | grep node3                                                                                                                                                                                                             

  831  kubectl describe pod longhorn-csi-plugin-5t9mg -n longhorn-system                                                                                                                                                                                                    

  832  kubectl get pod longhorn-csi-plugin-5t9mg -n longhorn-system -o jsonpath='{.spec.containers[*].name}'                                                                                                                                                                

  833  kubectl logs longhorn-csi-plugin-5t9mg -n longhorn-system -c longhorn-csi-plugin --tail=200                                                                                                                                                                          

  834  kubectl logs longhorn-csi-plugin-5t9mg -n longhorn-system -c node-driver-registrar --tail=200                                                                                                                                                                        

  835  kubectl logs longhorn-csi-plugin-5t9mg -n longhorn-system -c liveness-probe --tail=200                                                                                                                                                                               

  836  kubectl get pods -A -o wide | grep node3                                                                                                                                                                                                                             

  837  kubectl get svc -n longhorn-system                                                                                                                                                                                                                                   

  838  kubectl get endpoints -n longhorn-system longhorn-backend                                                                                                                                                                                                            

  839  kubectl exec -it -n longhorn-system longhorn-manager-v92r6 -- sh                                                                                                                                                                                                     

  840  kubectl logs longhorn-manager-v92r6 -n longhorn-system -c longhorn-manager --previous --tail=200                                                                                                                                                                     

  841  kubectl run nettest-node3   -n default   --image=busybox:1.36   --restart=Never   --overrides='
  842  {
  843    "apiVersion":"v1",
  844    "spec":{
  845      "nodeName":"node3",
  846      "containers":[
  847        {
  848          "name":"nettest",
  849          "image":"busybox:1.36",
  850          "command":["sh","-c","sleep 3600"]
  851        }
  852      ]
  853    }
  854  }'                                                                                                                                                                                                                                                                   

  855  kubectl exec -it nettest-node3 -n default -- sh                                                                                                                                                                                                                      

  856  kubectl get svc -n kube-system                                                                                                                                                                                                                                       

  857  kubectl get endpoints -n kube-system kube-dns                                                                                                                                                                                                                        

  858  kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide                                                                                                                                                                                                          

  859  kubectl get endpoints -n kube-system rke2-coredns-rke2-coredns                                                                                                                                                                                                       

  860  kubectl get svc,endpoints -n kube-system | grep rke2-coredns                                                                                                                                                                                                         

  861  kubectl delete pod kube-proxy-node3 -n kube-system                                                                                                                                                                                                                   

  862  kubectl delete pod rke2-canal-9bqb7 -n kube-system                                                                                                                                                                                                                   

  863  kubectl get svc,endpoints -n kube-system | grep rke2-coredns                                                                                                                                                                                                         

  864  kubectl get svc -n longhorn-system                                                                                                                                                                                                                                   

  865  kubectl get pods -A -o wide | grep node3                                                                                                                                                                                                                             

  866  kubectl describe pod rke2-coredns-rke2-coredns-6464f98784-5qb44 -n kube-system                                                                                                                                                                                       

  867  kubectl exec -it nettest-node3 -n default -- sh                                                                                                                                                                                                                      

  868  kubectl get nodes -o wide                                                                                                                                                                                                                                            

  869  kubectl describe node node3                                                                                                                                                                                                                                          

  870  kubectl get pods -n longhorn-system -o wide                                                                                                                                                                                                                          

  871  kubectl get no                                                                                                                                                                                                                                                       

  872  kubectl get pods -n longhorn-system -o wide                                                                                                                                                                                                                          

  873  kubectl describe pod longhorn-csi-plugin-5t9mg -n longhorn-system                                                                                                                                                                                                    

  874  kubectl logs longhorn-csi-plugin-5t9mg -n longhorn-system -c longhorn-csi-plugin --tail=100                                                                                                                                                                          

  875  kubectl logs longhorn-csi-plugin-5t9mg -n longhorn-system -c node-driver-registrar --tail=100                                                                                                                                                                        

  876  kubectl logs longhorn-csi-plugin-5t9mg -n longhorn-system -c liveness-probe --tail=100                                                                                                                                                                               

  877  kubectl get svc -n longhorn-system longhorn-backend                                                                                                                                                                                                                  

  878  kubectl get endpoints -n longhorn-system longhorn-backend                                                                                                                                                                                                            

  879  kubectl get endpointslice -n longhorn-system | grep longhorn-backend                                                                                                                                                                                                 

  880  kubectl exec -it nettest-node3 -- sh                                                                                                                                                                                                                                 

  881  kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide                                                                                                                                                                                                          

  882  kubectl get svc -n kube-system kube-dns                                                                                                                                                                                                                              

  883  kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100                                                                                                                                                                                                           

  884  kubectl logs -n kube-system kube-proxy-node3 --tail=200                                                                                                                                                                                                              

  885  kubectl logs -n kube-system rke2-canal-8gdd6 --tail=200                                                                                                                                                                                                              

  886  kubectl get svc -n kube-system -o wide                                                                                                                                                                                                                               

  887  kubectl get endpoints -n kube-system                                                                                                                                                                                                                                 

  888  kubectl get endpointslice -n kube-system                                                                                                                                                                                                                             

  889  kubectl get pods -n kube-system -o wide | grep coredns                                                                                                                                                                                                               

  890  kubectl describe pod -n kube-system rke2-coredns-rke2-coredns-6464f98784-5qb44                                                                                                                                                                                       

  891  kubectl exec -it nettest-node3 -- sh                                                                                                                                                                                                                                 

  892  kubectl delete pod -n kube-system rke2-canal-8gdd6                                                                                                                                                                                                                   

  893  kubectl get po -n kube-system                                                                                                                                                                                                                                        

  894  kubectl exec -it nettest-node3 -- sh                                                                                                                                                                                                                                 

  895  kubectl get nodes -o wide                                                                                                                                                                                                                                            

  896  kubectl get pods -n kube-system -o wide                                                                                                                                                                                                                              

  897  kubectl get pods -n longhorn-system -o wide                                                                                                                                                                                                                          

  898  kubectl get nodes -o wide                                                                                                                                                                                                                                            

  899  kubectl get pods -n kube-system -o wide                                                                                                                                                                                                                              

  900  kubectl get pods -n longhorn-system -o wide                                                                                                                                                                                                                          

  901  kubectl get pods -n kube-system -o wide                                                                                                                                                                                                                              

  902  kubectl exec -it nettest-node3 -- sh\                                                                                                                                                                                                                                

  903  kubectl exec -it nettest-node3 -- sh                                                                                                                                                                                                                                 

  904  kubectl exec -it nettest-node3 -- sh\                                                                                                                                                                                                                                

  905  kubectl get pods -n kube-system -o wide                                                                                                                                                                                                                              

  906  kubectl cordon node3                                                                                                                                                                                                                                                 

  907  kubectl drain node3 --ignore-daemonsets --delete-emptydir-data --force                                                                                                                                                                                               

  908  kubectl delete node node3                                                                                                                                                                                                                                            

  909  kubectl get nodes -o wide                                                                                                                                                                                                                                            

  910  cat /var/lib/rancher/rke2/server/node-token                                                                                                                                                                                                                          

  911  kubectl get nodes -o wide                                                                                                                                                                                                                                            

  912  kubectl describe node node3                                                                                                                                                                                                                                          

  913  kubectl get pods -n kube-system -o wide                                                                                                                                                                                                                              

  914  kubectl get pods -n longhorn-system -o wide                                                                                                                                                                                                                          

  915  kubectl get pods -n kube-system -o wide                                                                                                                                                                                                                              

  916  kubectl get pods -n longhorn-system -o wide                                                                                                                                                                                                                          

  917  kubectl get pods -n kube-system -o wide                                                                                                                                                                                                                              

  918  kubectl get pods -n longhorn-system -o wide                                                                                                                                                                                                                          

  919  kubectl describe pod longhorn-csi-plugin-qwp99 -n longhorn-system                                                                                                                                                                                                    

  920  kubectl logs longhorn-csi-plugin-qwp99 -n longhorn-system -c longhorn-csi-plugin --tail=100                                                                                                                                                                          

  921  kubectl logs longhorn-csi-plugin-qwp99 -n longhorn-system -c node-driver-registrar --tail=100                                                                                                                                                                        

  922  kubectl logs longhorn-csi-plugin-qwp99 -n longhorn-system -c longhorn-liveness-probe --tail=100                                                                                                                                                                      

  923  kubectl run nettest-node3 --image=busybox:1.36 --restart=Never --overrides='
  924  {
  925    "apiVersion": "v1",
  926    "spec": {
  927      "nodeName": "node3",
  928      "containers": [
  929        {
  930          "name": "busybox",
  931          "image": "busybox:1.36",
  932          "command": ["sh", "-c", "sleep 3600"]
  933        }
  934      ]
  935    }
  936  }'                                                                                                                                                                                                                                                                   

  937  kubectl exec -it nettest-node3 -- sh                                                                                                                                                                                                                                 

  938  kubectl get svc -n kube-system rke2-coredns-rke2-coredns                                                                                                                                                                                                             

  939  kubectl get endpoints -n kube-system rke2-coredns-rke2-coredns                                                                                                                                                                                                       

  940  kubectl get pods -n kube-system -o wide | grep coredns                                                                                                                                                                                                               

  941  kubectl describe pod -n kube-system rke2-coredns-rke2-coredns-6464f98784-5qb44                                                                                                                                                                                       

  942  kubectl describe pod -n kube-system rke2-coredns-rke2-coredns-autoscaler-67bb49dff-wxtwb                                                                                                                                                                             

  943  kubectl describe pod -n kube-system rke2-metrics-server-75d485c65b-x4tdl                                                                                                                                                                                             

  944  kubectl describe pod -n kube-system rke2-snapshot-controller-696989ffdd-2gf64                                                                                                                                                                                        

  945  kubectl describe pod -n kube-system rke2-canal-qx89q                                                                                                                                                                                                                 

  946  kubectl delete pod -n kube-system rke2-canal-wbqnp                                                                                                                                                                                                                   

  947  kubectl get pod -n kube-system                                                                                                                                                                                                                                       

  948  kubectl exec -it nettest-node3 -- sh                                                                                                                                                                                                                                 

  949  kubectl delete pod -n longhorn-system longhorn-csi-plugin-qwp99                                                                                                                                                                                                      

  950  kubectl get pods -n longhorn-system -o wide -w                                                                                                                                                                                                                       

  951  kubectl get pods -n longhorn-system -o wide                                                                                                                                                                                                                          

  952  kubectl delete pod -n longhorn-system longhorn-manager-k8x4h                                                                                                                                                                                                         

  953  kubectl rollout restart deployment/longhorn-ui -n longhorn-system                                                                                                                                                                                                    

  954  kubectl get no -n longhorn-system                                                                                                                                                                                                                                    

  955  kubectl get po -n longhorn-system                                                                                                                                                                                                                                    

  956  kubectl get po -n longhorn-system\                                                                                                                                                                                                                                   

  957  kubectl get po -n longhorn-system                                                                                                                                                                                                                                    

  958  kubectl edit nodes.longhorn.io node3 -n longhorn-system                                                                                                                                                                                                              

  959  kubectl get nodes.longhorn.io node3 -n longhorn-system -o yaml                                                                                                                                                                                                       

  960  kubectl delete pod -n longhorn-system longhorn-manager-l9fzs                                                                                                                                                                                                         

  961  kubectl get po -n longhorn-system                                                                                                                                                                                                                                    

  962  kubectl delete nodes.longhorn.io node3 -n longhorn-system                                                                                                                                                                                                            

  963  kubectl delete pod -n longhorn-system longhorn-manager-8n5qq                                                                                                                                                                                                         

  964  kubectl delete nodes.longhorn.io node3 -n longhorn-system                                                                                                                                                                                                            

  965  kubectl delete pod -n longhorn-system longhorn-manager-8n5qq                                                                                                                                                                                                         

  966  kubectl get nodes.longhorn.io node3 -n longhorn-system -o yaml                                                                                                                                                                                                       

  967  kubectl get po -n longhorn-system                                                                                                                                                                                                                                    

  968  kubectl get engines.longhorn.io -n longhorn-system -o yaml | grep -n -C5 "node3"                                                                                                                                                                                     

  969  kubectl get replicas.longhorn.io -n longhorn-system -o yaml | grep -n -C5 "node3"                                                                                                                                                                                    

  970  kubectl get volumeattachments.longhorn.io -n longhorn-system -o yaml | grep -n -C5 "node3"                                                                                                                                                                           

  971  kubectl get pv pvc-b45aa67b-8948-4901-8d8b-44ecc4afbd73 -o jsonpath='{.spec.claimRef.namespace}{" "}{.spec.claimRef.name}{"\n"}'                                                                                                                                     

  972  kubectl describe pvc -n database data-postgres-sts-0                                                                                                                                                                                                                 

  973  kubectl get engines.longhorn.io -n longhorn-system -o yaml | grep -n -C3 "pvc-b45aa67b-8948-4901-8d8b-44ecc4afbd73"                                                                                                                                                  

  974  kubectl get volumeattachments.longhorn.io -n longhorn-system -o yaml | grep -n -C3 "pvc-b45aa67b-8948-4901-8d8b-44ecc4afbd73"                                                                                                                                        

  975  kubectl delete replicas.longhorn.io pvc-0c9e60ba-c8f2-4c2a-9e8e-b177bd16462c-r-02c8bc33 -n longhorn-system                                                                                                                                                           

  976  kubectl delete nodes.longhorn.io node3 -n longhorn-system                                                                                                                                                                                                            

  977  kubectl delete pod -n longhorn-system -l app=longhorn-manager --field-selector spec.nodeName=node3                                                                                                                                                                   

  978  kubectl cordon node3                                                                                                                                                                                                                                                 

  979  kubectl scale sts -n database postgres-sts --replicas=0                                                                                                                                                                                                              

  980  kubectl get engines.longhorn.io -n longhorn-system -o yaml | grep -n -C3 "pvc-b45aa67b-8948-4901-8d8b-44ecc4afbd73"                                                                                                                                                  

  981  kubectl get volumeattachments.longhorn.io -n longhorn-system -o yaml | grep -n -C3 "pvc-b45aa67b-8948-4901-8d8b-44ecc4afbd73"                                                                                                                                        

  982  kubectl delete nodes.longhorn.io node3 -n longhorn-system                                                                                                                                                                                                            

  983  kubectl patch nodes.longhorn.io node3 -n longhorn-system --type merge -p '
  984  spec:
  985    allowScheduling: false
  986    disks:
  987      default-disk-fd0000000000:
  988        allowScheduling: false
  989  '

  990  kubectl get nodes.longhorn.io node3 -n longhorn-system -o yaml                                                                                                                                                                                                       

  991  kubectl delete nodes.longhorn.io node3 -n longhorn-system                                                                                                                                                                                                            

  992  kubectl delete pod -n longhorn-system -l app=longhorn-manager --field-selector spec.nodeName=node3
  993  sleep 10
  994  kubectl delete nodes.longhorn.io node3 -n longhorn-system                                                                                                                                                                                                            

  995  kubectl patch nodes.longhorn.io node3 -n longhorn-system --type='json' -p='[
  996    {"op":"remove","path":"/spec/disks/default-disk-fd0000000000"}
  997  ]'

  998  kubectl edit nodes.longhorn.io node3 -n longhorn-system                                                                                                                                                                                                              

  999  kubectl delete pod -n longhorn-system -l app=longhorn-manager --field-selector spec.nodeName=node3                                                                                                                                                                   

 1000  kubectl get nodes.longhorn.io node3 -n longhorn-system -o yaml                                                                                                                                                                                                       

 1001  kubectl get po
 1002  kubectl get all -n database
 1003  kubectl get ns
 1004  kubectl get po -n longhorn-system
 1005  kubectl uncordon node3
 1006  kubectl patch nodes.longhorn.io node3 -n longhorn-system --type merge -p '
spec:
  allowScheduling: true
  disks:
    default-disk-fd0000000000:
      allowScheduling: true
'
 1007  history
