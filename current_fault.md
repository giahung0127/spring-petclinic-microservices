**Lệnh cần thực hiện để fix port mismatch cho vets và visits service**

1. **Apply lại manifests đã chỉnh port:**
```bash
kubectl apply -f k8s-manifests/vets-service-deployment.yaml -n petclinic
kubectl apply -f k8s-manifests/visits-service-deployment.yaml -n petclinic
```

2. **Đợi rollout xong:**
```bash
kubectl rollout status deploy/vets-service -n petclinic
kubectl rollout status deploy/visits-service -n petclinic
```

3. **Test lại từ api-gateway với port 8080:**
```bash
GATEWAY_POD=$(kubectl get pod -n petclinic -l app=api-gateway -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n petclinic $GATEWAY_POD -c api-gateway -- curl -v http://vets-service.petclinic.svc.cluster.local:8080/vets
kubectl exec -n petclinic $GATEWAY_POD -c api-gateway -- curl -v "http://visits-service.petclinic.svc.cluster.local:8080/owners/*/pets/{petId}/visits"
```

**Lưu ý**: Đã sửa port từ 8083/8082 → 8080 cho cả vets-service và visits-service để khớp với port thực tế mà Spring Boot app đang lắng nghe (8080 mặc định).
