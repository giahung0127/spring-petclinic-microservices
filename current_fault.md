giahung@devops:~/spring-petclinic-microservices/k8s-manifests$ GATEWAY_POD=$(kubectl get pod -n petclinic -l app=api-gateway -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n petclinic $GATEWAY_POD -c istio-proxy -- curl -v http://customers-service.petclinic.svc.cluster.local:8081/customers
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Host customers-service.petclinic.svc.cluster.local:8081 was resolved.
* IPv6: (none)
* IPv4: 10.96.249.90
*   Trying 10.96.249.90:8081...
* Connected to customers-service.petclinic.svc.cluster.local (10.96.249.90) port 8081
> GET /customers HTTP/1.1
> Host: customers-service.petclinic.svc.cluster.local:8081
> User-Agent: curl/8.5.0
> Accept: */*
> 
* Recv failure: Connection reset by peer
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
* Closing connection
curl: (56) Recv failure: Connection reset by peer
command terminated with exit code 56
giahung@devops:~/spring-petclinic-microservices/k8s-manifests$ 


liệu có phải liên tục sai là do setting mạng ở trên vm ko, nhưng mà hiện tại vm của t vẫn kết nối được thông qua NAT mà nhỉ?

---

**Phân tích sự cố**

- **Hiện tượng**: Từ `api-gateway` (container `istio-proxy`) gọi nội bộ đến  
  `http://customers-service.petclinic.svc.cluster.local:8081/customers`:
  - DNS resolve OK (`10.96.249.90`)
  - TCP connect OK (`Connected to ... port 8081`)
  - Nhưng **Recv failure: Connection reset by peer** → phía server (pod `customers-service`) hoặc sidecar của nó chủ động reset kết nối.

- **Kết luận quan trọng**:  
  Lỗi **không liên quan tới NAT của VM**:
  - Lệnh đang chạy bên trong pod `api-gateway` (`kubectl exec ...`) → traffic chỉ đi **trong cluster** (overlay network, CNI, Istio), không đi ra ngoài qua NAT của VM.
  - Nếu NAT VM có vấn đề thì những truy cập **ra ngoài Internet** mới bị ảnh hưởng; ở đây ta đang gọi **ClusterIP service** bên trong.

---

**Khả năng nguyên nhân**

1. **Ứng dụng `customers-service` lỗi/crash khi xử lý `/customers`**
   - Pod vẫn accept connection nhưng process bên trong bị lỗi và đóng kết nối đột ngột → `connection reset by peer`.

2. **Sidecar Istio của `customers-service` reset do cấu hình mesh**
   - DestinationRule/VirtualService cấu hình sai (mTLS, subset, host, port, policy, retry, v.v.).
   - Một số rule (có thể trong `retry-policy.yaml` hoặc file khác) có thể đang can thiệp vào traffic `/customers`.

3. **Service/port không khớp với port thực tế trong container**
   - Ví dụ: Service expose `8081` nhưng container app chỉ listen `8080` hoặc path mapping sai → Envoy/iptables vẫn accept nhưng không forward được đúng → reset.

---

**Các bước kiểm tra & khắc phục đề xuất**

1. **Kiểm tra trạng thái pod `customers-service`**
   - Chạy:
     - `kubectl get pods -n petclinic -l app=customers-service -o wide`
	 giahung@devops:~/spring-petclinic-microservices$ kubectl get pods -n petclinic -l app=customers-service -o wide
NAME                                 READY   STATUS    RESTARTS      AGE   IP            NODE                              NOMINATED NODE   READINESS GATES
customers-service-74cbc7fb45-54hn5   2/2     Running   4 (28m ago)   43h   10.244.0.4    petclinic-cluster-control-plane   <none>           <none>
customers-service-74cbc7fb45-vwhg8   2/2     Running   4 (28m ago)   43h   10.244.0.18   petclinic-cluster-control-plane   <none>           <none>
giahung@devops:~/spring-petclinic-microservices$ 


2. **Curl trực tiếp từ trong pod `customers-service` (bỏ qua Istio gateway)**
   - Lấy tên pod:
     - `CUST_POD=$(kubectl get pod -n petclinic -l app=customers-service -o jsonpath='{.items[0].metadata.name}')`
   - Exec vào container app:
     - `kubectl exec -n petclinic $CUST_POD -c customers-service -- curl -v http://localhost:8080/customers`
   - Kịch bản:
     - Nếu **vẫn bị reset/500** → lỗi thuộc về **ứng dụng `customers-service`** (code, cấu hình Spring, DB, v.v.).
     - Nếu **trả JSON bình thường** → app OK, lỗi nằm ở **Istio routing/cấu hình service/port**.
	 giahung@devops:~/spring-petclinic-microservices$ kubectl exec -n petclinic $CUST_POD -c customers-service -- curl -v http://localhost:8081/customers
* Host localhost:8081 was resolved.
* IPv6: ::1
* IPv4: 127.0.0.1
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0*   Trying [::1]:8081...
* connect to ::1 port 8081 from ::1 port 41926 failed: Connection refused
*   Trying 127.0.0.1:8081...
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* connect to 127.0.0.1 port 8081 from 127.0.0.1 port 45030 failed: Connection refused
* Failed to connect to localhost port 8081 after 19 ms: Couldn't connect to server
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
* Closing connection
curl: (7) Failed to connect to localhost port 8081 after 19 ms: Couldn't connect to server
command terminated with exit code 7
giahung@devops:~/spring-petclinic-microservices$ 


3. **Kiểm tra Service & port mapping của `customers-service`**
   - `kubectl get svc -n petclinic customers-service -o yaml`
   - Đối chiếu:
     - `spec.ports[*].port` (8081?)  
     - `spec.ports[*].targetPort` có trùng với port mà container app đang listen không (ví dụ 8081/8080).
   - Nếu sai → sửa manifest service trong `k8s-manifests` cho khớp với cấu hình app (Spring `server.port`).

4. **Kiểm tra cấu hình Istio (DestinationRule / VirtualService / Policy) liên quan `customers-service`**
   - Tìm trong `k8s-manifests` (vd: `retry-policy.yaml`, các file `*-customers*.yaml`):
     - Host phải đúng: `customers-service.petclinic.svc.cluster.local` hoặc `customers-service` (cùng namespace).
     - Port trong `route.destination.port.number` phải khớp với service.
     - Nếu bật mTLS:
       - DestinationRule phải khớp mode (UNSET, DISABLE, ISTIO_MUTUAL, v.v.).
   - Thử **tạm thời apply manifest tối giản** (không retry/fault injection) để xem còn bị reset không:
     - Nếu hết lỗi → cấu hình policy trước đó là nguyên nhân.

	*Thực hiện: kiểm tra trong thư mục k8s-manifests của project này đi 


---

**Kết quả kiểm tra và hướng xử lý**

- Log curl bên trong pod `customers-service` báo `Connection refused` trên `localhost:8081` → ứng dụng **không lắng nghe 8081**.
- Kiểm tra `spring-petclinic-customers-service/src/main/resources/application.yml` **không đặt `server.port`**, Spring Boot mặc định 8080.
- Kiểm tra `k8s-manifests/customers-service-deployment.yaml`:  
  - `containerPort: 8081` và Service `port/targetPort: 8081` → **sai port**.
- Đã sửa manifest để dùng **8080** cho cả containerPort và Service (file `k8s-manifests/customers-service-deployment.yaml`).

**Cần làm trên cluster**

1) Apply lại manifest đã chỉnh:
   - `kubectl apply -f k8s-manifests/customers-service-deployment.yaml -n petclinic`

2) Chờ pod rollout:
   - `kubectl rollout status deploy/customers-service -n petclinic`

3) Kiểm tra lại:
   - `CUST_POD=$(kubectl get pod -n petclinic -l app=customers-service -o jsonpath='{.items[0].metadata.name}')`
   - `kubectl exec -n petclinic $CUST_POD -c customers-service -- curl -v http://localhost:8080/customers` (lưu ý đổi sang 8080)

4) Gọi lại từ `api-gateway`:
   - `GATEWAY_POD=$(kubectl get pod -n petclinic -l app=api-gateway -o jsonpath='{.items[0].metadata.name}')`
   - `kubectl exec -n petclinic $GATEWAY_POD -c istio-proxy -- curl -v http://customers-service.petclinic.svc.cluster.local:8080/customers`

Nếu trả 200/JSON thì lỗi đã hết; nếu còn reset thì kiểm tra tiếp Istio policies.

5. **Thử gọi lại từ `api-gateway` sau khi sửa**
   - Chạy lại chuỗi lệnh:
     - `GATEWAY_POD=$(kubectl get pod -n petclinic -l app=api-gateway -o jsonpath='{.items[0].metadata.name}')`
     - `kubectl exec -n petclinic $GATEWAY_POD -c istio-proxy -- curl -v http://customers-service.petclinic.svc.cluster.local:8080/customers`
   - Mong đợi:
     - HTTP/1.1 200
     - Body JSON list customers.

	 giahung@devops:~/spring-petclinic-microservices$ GATEWAY_POD=$(kubectl get pod -n petclinic -l app=api-gateway -o jsonpath='{.items[0].metadata.name}')
giahung@devops:~/spring-petclinic-microservices$ kubectl exec -n petclinic $GATEWAY_POD -c istio-proxy -- curl -v http://customers-service.petclinic.svc.cluster.local:8081/customers
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Host customers-service.petclinic.svc.cluster.local:8081 was resolved.
* IPv6: (none)
* IPv4: 10.96.249.90
*   Trying 10.96.249.90:8081...
* Connected to customers-service.petclinic.svc.cluster.local (10.96.249.90) port 8081
> GET /customers HTTP/1.1
> Host: customers-service.petclinic.svc.cluster.local:8081
> User-Agent: curl/8.5.0
> Accept: */*
> 
* Recv failure: Connection reset by peer
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
* Closing connection
curl: (56) Recv failure: Connection reset by peer
command terminated with exit code 56
giahung@devops:~/spring-petclinic-microservices$ 

---

**Tóm tắt trả lời câu hỏi ở dòng 23**

- **Không**: lỗi lặp đi lặp lại này **không phải** do setting mạng/NAT trên VM.
- **Đúng hơn**: nó xuất phát từ **bên trong cluster**:
  - hoặc do **ứng dụng `customers-service`** bị lỗi,
  - hoặc do **cấu hình Service/port/Istio** cho `customers-service` chưa đúng.

Sau khi cậu chạy xong các bước 1–4 phía trên, nếu cần thì paste thêm log/manifest, tớ sẽ giúp soi cụ thể hơn xem dòng nào sai.