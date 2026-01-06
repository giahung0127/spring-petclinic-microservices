**Checklist lệnh cần chạy để debug tiếp lỗi `connection reset`**

1. **Kiểm tra Service & Endpoints của `customers-service`**
   - `kubectl get svc -n petclinic customers-service -o yaml`
   - `kubectl get ep -n petclinic customers-service`
   giahung@devops:~$ kubectl get svc -n petclinic customers-service -o yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"customers-service","namespace":"petclinic"},"spec":{"ports":[{"port":8080,"targetPort":8080}],"selector":{"app":"customers-service"}}}
  creationTimestamp: "2026-01-04T05:18:55Z"
  name: customers-service
  namespace: petclinic
  resourceVersion: "130318"
  uid: 9eca15f6-84c1-4c55-8001-ad4ef2eb07a9
spec:
  clusterIP: 10.96.249.90
  clusterIPs:
  - 10.96.249.90
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: customers-service
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
giahung@devops:~$ kubectl get ep -n petclinic customers-service
NAME                ENDPOINTS                           AGE
customers-service   10.244.0.14:8080,10.244.0.17:8080   2d4h
giahung@devops:~$ 


2. **Kiểm tra pod và log app `customers-service`**
   - `kubectl get pods -n petclinic -l app=customers-service -o wide`
   - `kubectl logs -n petclinic -l app=customers-service -c customers-service`
   giahung@devops:~$ kubectl get pods -n petclinic -l app=customers-service -o wide
NAME                                 READY   STATUS    RESTARTS      AGE   IP            NODE                              NOMINATED NODE   READINESS GATES
customers-service-74cbc7fb45-54hn5   2/2     Running   6 (12m ago)   44h   10.244.0.17   petclinic-cluster-control-plane   <none>           <none>
customers-service-74cbc7fb45-vwhg8   2/2     Running   6 (12m ago)   44h   10.244.0.14   petclinic-cluster-control-plane   <none>           <none>
customers-service-75d87dbc66-8cqc4   0/2     Pending   0             41m   <none>        <none>                            <none>           <none>
giahung@devops:~$ kubectl logs -n petclinic -l app=customers-service -c customers-service
	at com.netflix.discovery.DiscoveryClient.getAndStoreFullRegistry(DiscoveryClient.java:1046)
	at com.netflix.discovery.DiscoveryClient.fetchRegistry(DiscoveryClient.java:961)
	at com.netflix.discovery.DiscoveryClient.refreshRegistry(DiscoveryClient.java:1473)
	at com.netflix.discovery.DiscoveryClient$CacheRefreshThread.run(DiscoveryClient.java:1441)
	at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:539)
	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1136)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:635)
	at java.base/java.lang.Thread.run(Thread.java:840)

	at com.netflix.discovery.shared.transport.decorator.SessionedEurekaHttpClient.execute(SessionedEurekaHttpClient.java:76)
	at com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator.sendHeartBeat(EurekaHttpClientDecorator.java:89)
	at com.netflix.discovery.DiscoveryClient.renew(DiscoveryClient.java:845)
	at com.netflix.discovery.DiscoveryClient$HeartbeatThread.run(DiscoveryClient.java:1402)
	at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:539)
	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1136)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:635)
	at java.base/java.lang.Thread.run(Thread.java:840)

giahung@devops:~$ 

3. **Kiểm tra lại curl nội bộ (đã OK nhưng để đối chiếu)**
   - Lấy pod:  
     `CUST_POD=$(kubectl get pod -n petclinic -l app=customers-service -o jsonpath='{.items[0].metadata.name}')`
   - Gọi trực tiếp app:  
     `kubectl exec -n petclinic $CUST_POD -c customers-service -- curl -v http://localhost:8080/owners`
     giahung@devops:~$ CUST_POD=$(kubectl get pod -n petclinic -l app=customers-service -o jsonpath='{.items[0].metadata.name}')
giahung@devops:~$ kubectl exec -n petclinic $CUST_POD -c customers-service -- curl -v http://localhost:8080/owners
* Host localhost:8080 was resolved.
* IPv6: ::1
* IPv4: 127.0.0.1
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0*   Trying [::1]:8080...
* Connected to localhost (::1) port 8080
> GET /owners HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/8.5.0
> Accept: */*
> 
  0     0    0     0    0     0      0      0 --:--:--  0:00:15 --:--:--     0< HTTP/1.1 200 
< Content-Type: application/json
< Transfer-Encoding: chunked
< Date: Tue, 06 Jan 2026 09:58:49 GMT
< 
{ [2 bytes data]
100     2    0     2    0     0      0      0 --[]:--:--  0:00:15 --:--:--     0
* Connection #0 to host localhost left intact
giahung@devops:~$ 

4. **Kiểm tra log sidecar Istio của `customers-service`**
   - `kubectl logs -n petclinic -l app=customers-service -c istio-proxy`
   giahung@devops:~$ kubectl logs -n petclinic -l app=customers-service -c istio-proxy
2026-01-06T09:46:14.743074Z	info	xdsproxy	connected to delta upstream XDS server: istiod.istio-system.svc:15012	id=1
2026-01-06T09:46:14.850303Z	info	cache	generated new workload certificate	resourceName=default latency=5.454288475s ttl=23h59m59.149708198s
2026-01-06T09:46:14.850478Z	info	cache	Root cert has changed, start rotating root cert
2026-01-06T09:46:14.850938Z	info	cache	returned workload trust anchor from cache	ttl=23h59m59.149066425s
2026-01-06T09:46:14.973702Z	info	ads	ADS: new connection for node:1
2026-01-06T09:46:14.973887Z	info	cache	returned workload certificate from cache	ttl=23h59m59.02611722s
2026-01-06T09:46:14.984786Z	info	ads	ADS: new connection for node:2
2026-01-06T09:46:14.985505Z	info	cache	returned workload trust anchor from cache	ttl=23h59m59.014499783s
2026-01-06T09:46:16.755895Z	info	Readiness succeeded in 7.57582274s
2026-01-06T09:46:16.759205Z	info	Envoy proxy is ready
2026-01-06T09:46:13.534809Z	info	cache	generated new workload certificate	resourceName=default latency=1.576812613s ttl=23h59m59.465198185s
2026-01-06T09:46:13.537216Z	info	cache	Root cert has changed, start rotating root cert
2026-01-06T09:46:13.537889Z	info	cache	returned workload trust anchor from cache	ttl=23h59m59.462116954s
2026-01-06T09:46:13.660654Z	info	ads	ADS: new connection for node:1
2026-01-06T09:46:13.661651Z	info	cache	returned workload certificate from cache	ttl=23h59m59.338355131s
2026-01-06T09:46:13.732445Z	info	ads	ADS: new connection for node:2
2026-01-06T09:46:13.741560Z	info	cache	returned workload trust anchor from cache	ttl=23h59m59.258447008s
2026-01-06T09:46:14.839053Z	error	failed scraping envoy metrics: error scraping http://localhost:15090/stats/prometheus: Get "http://localhost:15090/stats/prometheus": dial tcp [::1]:15090: connect: connection refused
2026-01-06T09:46:15.731538Z	info	Readiness succeeded in 3.824065428s
2026-01-06T09:46:15.732073Z	info	Envoy proxy is ready
giahung@devops:~$ 

5. **Kiểm tra log sidecar Istio của `api-gateway` khi call lỗi**
   - Lấy pod gateway:  
     `GATEWAY_POD=$(kubectl get pod -n petclinic -l app=api-gateway -o jsonpath='{.items[0].metadata.name}')`
   - Xem log sidecar:  
     `kubectl logs -n petclinic $GATEWAY_POD -c istio-proxy`
     giahung@devops:~$ GATEWAY_POD=$(kubectl get pod -n petclinic -l app=api-gateway -o jsonpath='{.items[0].metadata.name}')
giahung@devops:~$ kubectl logs -n petclinic $GATEWAY_POD -c istio-proxy
2026-01-06T09:45:44.446544Z	info	FLAG: --concurrency="0"
2026-01-06T09:45:44.446625Z	info	FLAG: --domain="petclinic.svc.cluster.local"
2026-01-06T09:45:44.446632Z	info	FLAG: --help="false"
2026-01-06T09:45:44.446634Z	info	FLAG: --log_as_json="false"
2026-01-06T09:45:44.446636Z	info	FLAG: --log_caller=""
2026-01-06T09:45:44.446638Z	info	FLAG: --log_output_level="default:info"
2026-01-06T09:45:44.446640Z	info	FLAG: --log_stacktrace_level="default:none"
2026-01-06T09:45:44.446657Z	info	FLAG: --log_target="[stdout]"
2026-01-06T09:45:44.446659Z	info	FLAG: --meshConfig="./etc/istio/config/mesh"
2026-01-06T09:45:44.446661Z	info	FLAG: --outlierLogPath=""
2026-01-06T09:45:44.446664Z	info	FLAG: --profiling="true"
2026-01-06T09:45:44.446666Z	info	FLAG: --proxyComponentLogLevel="misc:error"
2026-01-06T09:45:44.446667Z	info	FLAG: --proxyLogLevel="warning"
2026-01-06T09:45:44.446669Z	info	FLAG: --serviceCluster="istio-proxy"
2026-01-06T09:45:44.446672Z	info	FLAG: --stsPort="0"
2026-01-06T09:45:44.446674Z	info	FLAG: --templateFile=""
2026-01-06T09:45:44.446675Z	info	FLAG: --tokenManagerPlugin=""
2026-01-06T09:45:44.446679Z	info	FLAG: --vklog="0"
2026-01-06T09:45:44.446682Z	info	Version 1.28.2-ab413ac6c1f40b2f7c69d97e0db4e712e4ef1ecc-Clean
2026-01-06T09:45:44.447255Z	info	Proxy role	ips=[10.244.0.8] type=sidecar id=api-gateway-9bd78f7f4-qpmh2.petclinic domain=petclinic.svc.cluster.local
2026-01-06T09:45:44.447567Z	info	Apply proxy config from env {}

2026-01-06T09:45:44.449746Z	info	cpu limit detected as 2, setting concurrency
2026-01-06T09:45:44.450305Z	info	Effective config: binaryPath: /usr/local/bin/envoy
concurrency: 2
configPath: ./etc/istio/proxy
controlPlaneAuthPolicy: MUTUAL_TLS
discoveryAddress: istiod.istio-system.svc:15012
drainDuration: 45s
proxyAdminPort: 15000
serviceCluster: istio-proxy
statNameLength: 189
statusPort: 15020
terminationDrainDuration: 5s

2026-01-06T09:45:44.450352Z	info	JWT policy is third-party-jwt
2026-01-06T09:45:44.450357Z	info	using credential fetcher of JWT type in cluster.local trust domain
2026-01-06T09:45:44.473213Z	info	Starting default Istio SDS Server
2026-01-06T09:45:44.474269Z	info	CA Endpoint istiod.istio-system.svc:15012, provider Citadel
2026-01-06T09:45:44.475552Z	info	Opening status port 15020
2026-01-06T09:45:44.475898Z	info	Using CA istiod.istio-system.svc:15012 cert with certs: var/run/secrets/istio/root-cert.pem
2026-01-06T09:45:44.485931Z	info	xdsproxy	Initializing with upstream address "istiod.istio-system.svc:15012" and cluster "Kubernetes"
2026-01-06T09:45:44.495499Z	info	Pilot SAN: [istiod.istio-system.svc]
2026-01-06T09:45:44.499276Z	info	sds	Starting SDS grpc server
2026-01-06T09:45:44.499901Z	info	sds	Starting SDS server for workload certificates, will listen on "var/run/secrets/workload-spiffe-uds/socket"
2026-01-06T09:45:44.504883Z	info	Starting proxy agent
2026-01-06T09:45:44.506094Z	info	Envoy command: [-c etc/istio/proxy/envoy-rev.json --drain-time-s 45 --drain-strategy immediate --local-address-ip-version v4 --file-flush-interval-msec 1000 --disable-hot-restart --allow-unknown-static-fields -l warning --component-log-level misc:error --skip-deprecated-logs --concurrency 2]
2026-01-06T09:45:54.501872Z	warn	ca	ca request failed, starting attempt 1 in 100.102703ms
2026-01-06T09:45:54.603652Z	warn	ca	ca request failed, starting attempt 2 in 196.282689ms
2026-01-06T09:45:54.800949Z	warn	ca	ca request failed, starting attempt 3 in 378.187649ms
2026-01-06T09:45:55.181331Z	warn	ca	ca request failed, starting attempt 4 in 838.708401ms
2026-01-06T09:45:56.022990Z	error	citadelclient	failed to sign CSR: create certificate: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing: dial tcp: lookup istiod.istio-system.svc on 10.96.0.10:53: read udp 10.244.0.8:32934->10.96.0.10:53: read: connection refused"
2026-01-06T09:45:56.025270Z	info	citadelclient	recreated connection
2026-01-06T09:45:56.025764Z	error	cache	resource:default failed to sign: create certificate: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing: dial tcp: lookup istiod.istio-system.svc on 10.96.0.10:53: read udp 10.244.0.8:32934->10.96.0.10:53: read: connection refused"
2026-01-06T09:45:56.025777Z	warn	sds	failed to warm certificate: failed to generate workload certificate: create certificate: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing: dial tcp: lookup istiod.istio-system.svc on 10.96.0.10:53: read udp 10.244.0.8:32934->10.96.0.10:53: read: connection refused"
2026-01-06T09:45:57.304799Z	error	failed scraping envoy metrics: error scraping http://localhost:15090/stats/prometheus: Get "http://localhost:15090/stats/prometheus": dial tcp [::1]:15090: connect: connection refused
2026-01-06T09:46:00.547449Z	warn	ca	ca request failed, starting attempt 1 in 108.576905ms
2026-01-06T09:46:00.657484Z	warn	ca	ca request failed, starting attempt 2 in 189.922046ms
2026-01-06T09:46:00.851401Z	warn	ca	ca request failed, starting attempt 3 in 421.068855ms
2026-01-06T09:46:01.273198Z	warn	ca	ca request failed, starting attempt 4 in 755.2412ms
2026-01-06T09:46:02.029912Z	error	citadelclient	failed to sign CSR: create certificate: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing: dial tcp: lookup istiod.istio-system.svc on 10.96.0.10:53: read udp 10.244.0.8:39657->10.96.0.10:53: read: connection refused"
2026-01-06T09:46:02.035404Z	info	citadelclient	recreated connection
2026-01-06T09:46:02.035564Z	error	cache	resource:default failed to sign: create certificate: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing: dial tcp: lookup istiod.istio-system.svc on 10.96.0.10:53: read udp 10.244.0.8:39657->10.96.0.10:53: read: connection refused"
2026-01-06T09:46:02.035570Z	warn	sds	failed to warm certificate: failed to generate workload certificate: create certificate: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing: dial tcp: lookup istiod.istio-system.svc on 10.96.0.10:53: read udp 10.244.0.8:39657->10.96.0.10:53: read: connection refused"
2026-01-06T09:46:11.291809Z	warn	ca	ca request failed, starting attempt 1 in 108.066249ms
2026-01-06T09:46:11.433962Z	warn	ca	ca request failed, starting attempt 2 in 204.78948ms
2026-01-06T09:46:11.654399Z	warn	ca	ca request failed, starting attempt 3 in 391.444178ms
2026-01-06T09:46:12.047577Z	warn	ca	ca request failed, starting attempt 4 in 872.604502ms
2026-01-06T09:46:12.921627Z	error	citadelclient	failed to sign CSR: create certificate: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing: dial tcp 10.96.99.103:15012: connect: connection refused"
2026-01-06T09:46:12.922342Z	info	citadelclient	recreated connection
2026-01-06T09:46:12.922421Z	error	cache	resource:default failed to sign: create certificate: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing: dial tcp 10.96.99.103:15012: connect: connection refused"
2026-01-06T09:46:12.922429Z	warn	sds	failed to warm certificate: failed to generate workload certificate: create certificate: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing: dial tcp 10.96.99.103:15012: connect: connection refused"
2026-01-06T09:46:13.328035Z	info	xdsproxy	connected to delta upstream XDS server: istiod.istio-system.svc:15012	id=4
2026-01-06T09:46:13.971092Z	info	ads	ADS: new connection for node:1
2026-01-06T09:46:14.028702Z	info	ads	ADS: new connection for node:2
2026-01-06T09:46:14.822509Z	error	failed scraping envoy metrics: error scraping http://localhost:15090/stats/prometheus: Get "http://localhost:15090/stats/prometheus": dial tcp [::1]:15090: connect: connection refused
2026-01-06T09:46:14.859946Z	info	cache	generated new workload certificate	resourceName=default latency=888.349848ms ttl=23h59m59.140058841s
2026-01-06T09:46:14.866932Z	info	cache	Root cert has changed, start rotating root cert
2026-01-06T09:46:14.876904Z	info	cache	returned workload trust anchor from cache	ttl=23h59m59.123100987s
2026-01-06T09:46:14.877640Z	info	cache	returned workload certificate from cache	ttl=23h59m59.122364634s
2026-01-06T09:46:14.877843Z	info	cache	returned workload trust anchor from cache	ttl=23h59m59.122162188s
2026-01-06T09:46:14.932672Z	info	cache	returned workload trust anchor from cache	ttl=23h59m59.067332352s
2026-01-06T09:46:16.170238Z	info	Readiness succeeded in 31.222657874s
2026-01-06T09:46:16.175782Z	info	Envoy proxy is ready
giahung@devops:~$ 

6. **Xác nhận lại AuthorizationPolicy/mTLS đã apply đúng**
   - `kubectl get authorizationpolicy -n petclinic -o yaml`
   - `kubectl get peerauthentication -n petclinic -o yaml`
   giahung@devops:~$ kubectl get authorizationpolicy -n petclinic -o yaml
apiVersion: v1
items:
- apiVersion: security.istio.io/v1
  kind: AuthorizationPolicy
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"security.istio.io/v1beta1","kind":"AuthorizationPolicy","metadata":{"annotations":{},"name":"customers-service-policy","namespace":"petclinic"},"spec":{"action":"ALLOW","rules":[{"from":[{"source":{"principals":["cluster.local/ns/petclinic/sa/api-gateway"]}}],"to":[{"operation":{"methods":["GET","POST"]}}]}],"selector":{"matchLabels":{"app":"customers-service"}}}}
    creationTimestamp: "2026-01-04T06:23:10Z"
    generation: 3
    name: customers-service-policy
    namespace: petclinic
    resourceVersion: "93370"
    uid: 7a7fd879-1e2b-440b-8809-d74ca75a600b
  spec:
    action: ALLOW
    rules:
    - from:
      - source:
          principals:
          - cluster.local/ns/petclinic/sa/api-gateway
      to:
      - operation:
          methods:
          - GET
          - POST
    selector:
      matchLabels:
        app: customers-service
- apiVersion: security.istio.io/v1
  kind: AuthorizationPolicy
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"security.istio.io/v1beta1","kind":"AuthorizationPolicy","metadata":{"annotations":{},"name":"deny-all","namespace":"petclinic"},"spec":{"action":"DENY","rules":[{}]}}
    creationTimestamp: "2026-01-04T13:05:06Z"
    generation: 1
    name: deny-all
    namespace: petclinic
    resourceVersion: "117527"
    uid: 23305183-e38e-4adc-980b-2067dd04c0f1
  spec:
    action: DENY
    rules:
    - {}
- apiVersion: security.istio.io/v1
  kind: AuthorizationPolicy
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"security.istio.io/v1beta1","kind":"AuthorizationPolicy","metadata":{"annotations":{},"name":"vets-service-policy","namespace":"petclinic"},"spec":{"action":"ALLOW","rules":[{"from":[{"source":{"principals":["cluster.local/ns/petclinic/sa/api-gateway"]}}],"to":[{"operation":{"methods":["GET"]}}]}],"selector":{"matchLabels":{"app":"vets-service"}}}}
    creationTimestamp: "2026-01-04T06:23:10Z"
    generation: 3
    name: vets-service-policy
    namespace: petclinic
    resourceVersion: "93378"
    uid: 0d27d13f-be40-4606-a0cc-4898b0d0addc
  spec:
    action: ALLOW
    rules:
    - from:
      - source:
          principals:
          - cluster.local/ns/petclinic/sa/api-gateway
      to:
      - operation:
          methods:
          - GET
    selector:
      matchLabels:
        app: vets-service
- apiVersion: security.istio.io/v1
  kind: AuthorizationPolicy
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"security.istio.io/v1beta1","kind":"AuthorizationPolicy","metadata":{"annotations":{},"name":"visits-service-policy","namespace":"petclinic"},"spec":{"action":"ALLOW","rules":[{"from":[{"source":{"principals":["cluster.local/ns/petclinic/sa/api-gateway"]}}],"to":[{"operation":{"methods":["GET","POST"]}}]}],"selector":{"matchLabels":{"app":"visits-service"}}}}
    creationTimestamp: "2026-01-04T06:23:10Z"
    generation: 3
    name: visits-service-policy
    namespace: petclinic
    resourceVersion: "93386"
    uid: ed0fedf8-e072-4152-bd0b-9465f381a0de
  spec:
    action: ALLOW
    rules:
    - from:
      - source:
          principals:
          - cluster.local/ns/petclinic/sa/api-gateway
      to:
      - operation:
          methods:
          - GET
          - POST
    selector:
      matchLabels:
        app: visits-service
kind: List
metadata:
  resourceVersion: ""
giahung@devops:~$ kubectl get peerauthentication -n petclinic -o yaml
apiVersion: v1
items:
- apiVersion: security.istio.io/v1
  kind: PeerAuthentication
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"security.istio.io/v1beta1","kind":"PeerAuthentication","metadata":{"annotations":{},"name":"default","namespace":"petclinic"},"spec":{"mtls":{"mode":"STRICT"}}}
    creationTimestamp: "2026-01-04T06:10:03Z"
    generation: 1
    name: default
    namespace: petclinic
    resourceVersion: "83852"
    uid: 1bca5b96-efd8-4885-8762-cdcd7b39a5f9
  spec:
    mtls:
      mode: STRICT
kind: List
metadata:
  resourceVersion: ""
giahung@devops:~$ 

7. **Gọi lại qua mesh để tái hiện lỗi (sau khi đã xem logs phía trên)**
   - `kubectl exec -n petclinic $GATEWAY_POD -c istio-proxy -- curl -v http://customers-service.petclinic.svc.cluster.local:8080/owners`
   giahung@devops:~$ kubectl exec -n petclinic $GATEWAY_POD -c istio-proxy -- curl -v http://customers-service.petclinic.svc.cluster.local:8080/owners
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
* Recv failure: Connection reset by peer
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
* Closing connection
curl: (56) Recv failure: Connection reset by peer
command terminated with exit code 56
giahung@devops:~$ 

---

**Phân tích từ output đã thu thập**

- Service/Endpoints đúng 8080, pod app trả 200 khi gọi `localhost:8080/owners` → app OK.
- mTLS: PeerAuthentication STRICT, DestinationRule ISTIO_MUTUAL (file `mtls-destination-rule.yaml`) → hợp lệ.
- AuthorizationPolicy: đang có `deny-all` không selector → chặn toàn bộ request (đánh bại các ALLOW). Đây là nguyên nhân cao nhất cho `connection reset` khi đi qua mesh.
- Log istio-proxy của `api-gateway` có lỗi kết nối CA/istiod nhưng sau đó Ready; lỗi chính vẫn là chặn bởi policy.

**Sửa để phù hợp README/guide và cho phép traffic**

- Xóa policy `deny-all` (hoặc giới hạn selector). Đã chỉnh file `k8s-manifests/authorization-policy.yaml` để bỏ block `deny-all`.

**Lệnh cần chạy tiếp trên cluster**

1) Apply lại policy:
   - `kubectl apply -f k8s-manifests/authorization-policy.yaml -n petclinic`

2) Gọi lại kiểm tra qua mesh:
   - `GATEWAY_POD=$(kubectl get pod -n petclinic -l app=api-gateway -o jsonpath='{.items[0].metadata.name}')`
   - `kubectl exec -n petclinic $GATEWAY_POD -c istio-proxy -- curl -v http://customers-service.petclinic.svc.cluster.local:8080/owners`

3) Nếu vẫn lỗi, lấy log istio-proxy mới sau khi apply:
   - `kubectl logs -n petclinic -l app=customers-service -c istio-proxy --since=5m`
   - `kubectl logs -n petclinic $GATEWAY_POD -c istio-proxy --since=5m`
