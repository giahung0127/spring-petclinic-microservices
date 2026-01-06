**Kết luận cuối cùng & cách sửa để chạy được**

- Toàn bộ manifest hiện **đã khớp** README/guide:
  - `api-gateway`: port 8080.
  - `customers-service`: app chạy 8080, Service/Endpoints trỏ 8080.
  - mTLS STRICT + DestinationRule ISTIO_MUTUAL + AuthorizationPolicy ALLOW theo service account.
- Lý do vẫn bị `connection reset`: mình đang `kubectl exec ... -c istio-proxy` rồi `curl http://customers-service...` trực tiếp từ **container sidecar** trong khi namespace bật **mTLS STRICT**. Traffic này **không đi qua Envoy inbound/outbound path đúng chuẩn**, nên bị phía đích reset.

**Cách sửa lệnh curl (debug) cho đúng**

Thay vì exec vào `istio-proxy`, hãy exec vào **container app** `api-gateway` (Envoy sẽ tự chặn & nâng cấp HTTP → mTLS):

```bash
GATEWAY_POD=$(kubectl get pod -n petclinic -l app=api-gateway -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n petclinic $GATEWAY_POD -c api-gateway -- \
  curl -v http://customers-service.petclinic.svc.cluster.local:8080/owners
```

Nếu lệnh này trả về 200 thì toàn bộ config (k8s + Istio) đã đúng; lỗi trước giờ là do **cách mình test từ `istio-proxy` container**, không phải do setting trong project.


giahung@devops:~/spring-petclinic-microservices$ GATEWAY_POD=$(kubectl get pod -n petclinic -l app=api-gateway -o jsonpath='{.items[0].metadata.name}')
giahung@devops:~/spring-petclinic-microservices$ kubectl exec -n petclinic $GATEWAY_POD -c api-gateway -- \
  curl -v http://customers-service.petclinic.svc.cluster.local:8080/owners
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Host customers-service.petclinic.svc.cluster.local:8080 was resolved.
* IPv6: (none)
* IPv4: 10.96.249.90
*   Trying 10.96.249.90:8080...
* Connected to customers-service.petclinic.svc.cluster.local (10.96.249.90) port 8080
> GET /owners HTTP/1.1
> Host: customers-service.petclinic.svc.cluster.local:8080
> User-Agent: curl/8.5.0
> Accept: */*
> 
  0     0    0     0    0     0      0      0 --:--:--  0:00:09 --:--:--     0< HTTP/1.1 200 OK
< content-type: application/json
< date: Tue, 06 Jan 2026 11:38:45 GMT
< x-envoy-upstream-service-time: 9434
< server: envoy
< transfer-encoding: chunked
< 
{ [2 bytes data]
100     2    0     2    0     0      0      0 --:--:--  0:00:09 --:--:--     0
* Connection #0 to host customers-service.petclinic.svc.cluster.local left intact
giahung@devops:~/spring-petclinic-microservices$ 



nè output nè, trả về 200 z là đúng r đúng ko ?