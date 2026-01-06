**Test từ customers-service đến vets-service (bị chặn - đúng như mong đợi)**

```bash
CUSTOMERS_POD=$(kubectl get pod -n petclinic -l app=customers-service -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n petclinic $CUSTOMERS_POD -c istio-proxy -- curl -v http://vets-service.petclinic.svc.cluster.local:8083/vets
```

Output:
```
* Connected to vets-service.petclinic.svc.cluster.local (10.96.105.26) port 8083
> GET /vets HTTP/1.1
...
* Recv failure: Connection reset by peer
curl: (56) Recv failure: Connection reset by peer
```

**Phân tích**:
- Connection reset này có thể do:
  1. **Authorization Policy chặn** (đúng như mong đợi): `customers-service` không được phép gọi `vets-service` theo policy → đây là hành vi đúng.
  2. **Port không đúng**: Cần kiểm tra xem `vets-service` và `visits-service` có đang dùng port đúng không.

---

**Đã thực hiện**:

1. ✅ **Khôi phục `deny-all` policy** vào `k8s-manifests/authorization-policy.yaml`

2. **Kiểm tra port của vets và visits service**:

**Vets Service**:
- Deployment manifest: `containerPort: 8083`, Service `port/targetPort: 8083`
- Application.yml: Không có `server.port` → Spring Boot mặc định **8080**
- **Kết luận**: Có thể cần sửa giống customers-service (8083 → 8080), nhưng cần test trước.

**Visits Service**:
- Deployment manifest: `containerPort: 8082`, Service `port/targetPort: 8082`
- Application.yml: Không có `server.port` → Spring Boot mặc định **8080**
- **Kết luận**: Có thể cần sửa giống customers-service (8082 → 8080), nhưng cần test trước.

**Lệnh kiểm tra port thực tế**:

```bash
# Test vets-service từ trong pod của chính nó
VETS_POD=$(kubectl get pod -n petclinic -l app=vets-service -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n petclinic $VETS_POD -c vets-service -- curl -v http://localhost:8083/vets
kubectl exec -n petclinic $VETS_POD -c vets-service -- curl -v http://localhost:8080/vets

# Test visits-service từ trong pod của chính nó
VISITS_POD=$(kubectl get pod -n petclinic -l app=visits-service -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n petclinic $VISITS_POD -c visits-service -- curl -v http://localhost:8082/visits
kubectl exec -n petclinic $VISITS_POD -c visits-service -- curl -v http://localhost:8080/visits
```

**Lưu ý về lỗi connection reset từ customers-service đến vets-service**:
- Đây là **hành vi đúng** theo Authorization Policy: `customers-service` không được phép gọi `vets-service` (chỉ `api-gateway` được phép).
- Để test đúng, nên test từ `api-gateway`:
  ```bash
  GATEWAY_POD=$(kubectl get pod -n petclinic -l app=api-gateway -o jsonpath='{.items[0].metadata.name}')
  kubectl exec -n petclinic $GATEWAY_POD -c api-gateway -- curl -v http://vets-service.petclinic.svc.cluster.local:8083/vets
  ```
 