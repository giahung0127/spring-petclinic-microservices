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