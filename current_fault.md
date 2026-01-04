giahung@devops:~/spring-petclinic-microservices/k8s-manifests$ kubectl get pod $GATEWAY_POD -n petclinic -o jsonpath='{.spec.serviceAccountName}'
api-gatewaygiahung@devops:~/spring-petclinic-microservices/k8s-manifests$ kubectl get serviceaccounts -n petclinic
NAME                SECRETS   AGE
api-gateway         0         51m
customers-service   0         51m
default             0         47h
vets-service        0         51m
visits-service      0         51m
giahung@devops:~/spring-petclinic-microservices/k8s-manifests$ CUSTOMERS_POD=$(kubectl get pod -n petclinic -l app=customers-service -o jsonpath='{.items[0].metadata.name}')
kubectl logs -n petclinic $CUSTOMERS_POD -c istio-proxy --tail=50 | grep -i "denied\|rbac\|auth"
controlPlaneAuthPolicy: MUTUAL_TLS
giahung@devops:~/spring-petclinic-microservices/k8s-manifests$ kubectl logs -n petclinic $GATEWAY_POD -c istio-proxy --tail=50 | grep -i "denied\|rbac\|auth"
controlPlaneAuthPolicy: MUTUAL_TLS
giahung@devops:~/spring-petclinic-microservices/k8s-manifests$ CUSTOMERS_POD=$(kubectl get pod -n petclinic -l app=customers-service -o jsonpath='{.items[0].metadata.name}')
kubectl get pod $CUSTOMERS_POD -n petclinic -o jsonpath='{.spec.serviceAccountName}'
customers-servicegiahung@devops:~/spring-petclinic-microservicekubectl get authorizationpolicy -n petcliniclicy -n petclinic
NAME                       ACTION   AGE
customers-service-policy   ALLOW    65m
deny-all                   DENY     65m
vets-service-policy        ALLOW    65m
visits-service-policy      ALLOW    65m
giahung@devops:~/spring-petclinic-microservices/k8s-manifests$ kubectl logs -n petclinic $CUSTOMERS_POD -c istio-proxy --tail=100
2026-01-04T07:20:36.503082Z	info	FLAG: --concurrency="0"
2026-01-04T07:20:36.503156Z	info	FLAG: --domain="petclinic.svc.cluster.local"
2026-01-04T07:20:36.503277Z	info	FLAG: --help="false"
2026-01-04T07:20:36.503285Z	info	FLAG: --log_as_json="false"
2026-01-04T07:20:36.503287Z	info	FLAG: --log_caller=""
2026-01-04T07:20:36.503290Z	info	FLAG: --log_output_level="default:info"
2026-01-04T07:20:36.503292Z	info	FLAG: --log_stacktrace_level="default:none"
2026-01-04T07:20:36.503354Z	info	FLAG: --log_target="[stdout]"
2026-01-04T07:20:36.503360Z	info	FLAG: --meshConfig="./etc/istio/config/mesh"
2026-01-04T07:20:36.503363Z	info	FLAG: --outlierLogPath=""
2026-01-04T07:20:36.503365Z	info	FLAG: --profiling="true"
2026-01-04T07:20:36.503367Z	info	FLAG: --proxyComponentLogLevel="misc:error"
2026-01-04T07:20:36.503372Z	info	FLAG: --proxyLogLevel="warning"
2026-01-04T07:20:36.503374Z	info	FLAG: --serviceCluster="istio-proxy"
2026-01-04T07:20:36.503377Z	info	FLAG: --stsPort="0"
2026-01-04T07:20:36.503379Z	info	FLAG: --templateFile=""
2026-01-04T07:20:36.503383Z	info	FLAG: --tokenManagerPlugin=""
2026-01-04T07:20:36.503386Z	info	FLAG: --vklog="0"
2026-01-04T07:20:36.503389Z	info	Version 1.28.2-ab413ac6c1f40b2f7c69d97e0db4e712e4ef1ecc-Clean
2026-01-04T07:20:36.511831Z	info	Proxy role	ips=[10.244.0.38] type=sidecar id=customers-service-77c4bdbcb4-g55hd.petclinic domain=petclinic.svc.cluster.local
2026-01-04T07:20:36.514554Z	info	Apply proxy config from env {}

2026-01-04T07:20:36.533044Z	info	cpu limit detected as 2, setting concurrency
2026-01-04T07:20:36.534446Z	info	Effective config: binaryPath: /usr/local/bin/envoy
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

2026-01-04T07:20:36.536725Z	info	JWT policy is third-party-jwt
2026-01-04T07:20:36.536786Z	info	using credential fetcher of JWT type in cluster.local trust domain
2026-01-04T07:20:36.582854Z	info	Opening status port 15020
2026-01-04T07:20:36.582915Z	info	Starting default Istio SDS Server
2026-01-04T07:20:36.589845Z	info	CA Endpoint istiod.istio-system.svc:15012, provider Citadel
2026-01-04T07:20:36.590024Z	info	Using CA istiod.istio-system.svc:15012 cert with certs: var/run/secrets/istio/root-cert.pem
2026-01-04T07:20:36.617867Z	info	xdsproxy	Initializing with upstream address "istiod.istio-system.svc:15012" and cluster "Kubernetes"
2026-01-04T07:20:36.644040Z	info	sds	Starting SDS grpc server
2026-01-04T07:20:36.644228Z	info	sds	Starting SDS server for workload certificates, will listen on "var/run/secrets/workload-spiffe-uds/socket"
2026-01-04T07:20:36.656918Z	info	Pilot SAN: [istiod.istio-system.svc]
2026-01-04T07:20:36.661410Z	info	Starting proxy agent
2026-01-04T07:20:36.661532Z	info	Envoy command: [-c etc/istio/proxy/envoy-rev.json --drain-time-s 45 --drain-strategy immediate --local-address-ip-version v4 --file-flush-interval-msec 1000 --disable-hot-restart --allow-unknown-static-fields -l warning --component-log-level misc:error --skip-deprecated-logs --concurrency 2]
2026-01-04T07:20:37.278717Z	info	cache	generated new workload certificate	resourceName=default latency=657.388627ms ttl=23h59m59.721295309s
2026-01-04T07:20:37.278866Z	info	cache	Root cert has changed, start rotating root cert
2026-01-04T07:20:37.278919Z	info	cache	returned workload trust anchor from cache	ttl=23h59m59.721082388s
2026-01-04T07:20:37.460786Z	info	xdsproxy	connected to delta upstream XDS server: istiod.istio-system.svc:15012id=1
2026-01-04T07:20:37.789197Z	info	ads	ADS: new connection for node:1
2026-01-04T07:20:37.793332Z	info	cache	returned workload certificate from cache	ttl=23h59m59.206682567s
2026-01-04T07:20:37.810177Z	info	ads	ADS: new connection for node:2
2026-01-04T07:20:37.821101Z	info	cache	returned workload trust anchor from cache	ttl=23h59m59.178909835s
2026-01-04T07:20:39.365811Z	info	Readiness succeeded in 2.873469155s
2026-01-04T07:20:39.373737Z	info	Envoy proxy is ready
giahung@devops:~/spring-petclinic-microservices/k8s-manifests$ kubectl get authorizationpolicy deny-all -n petclinic -o yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"security.istio.io/v1beta1","kind":"AuthorizationPolicy","metadata":{"annotations":{},"name":"deny-all","namespace":"petclinic"},"spec":{"action":"DENY","rules":[{}]}}
  creationTimestamp: "2026-01-04T06:23:10Z"
  generation: 1
  name: deny-all
  namespace: petclinic
  resourceVersion: "85407"
  uid: 7b710760-87b8-4717-9caa-374798c1888e
spec:
  action: DENY
  rules:
  - {}
giahung@devops:~/spring-petclinic-microservices/k8s-manifests$ kubectl get deployment customers-service -n petclinic -o yaml | grep -A 5 serviceAccountName
      {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"name":"customers-service","namespace":"petclinic"},"spec":{"replicas":2,"selector":{"matchLabels":{"app":"customers-service"}},"template":{"metadata":{"labels":{"app":"customers-service"}},"spec":{"containers":[{"env":[{"name":"SPRING_PROFILES_ACTIVE","value":"native"}],"image":"springcommunity/spring-petclinic-customers-service:latest","name":"customers-service","ports":[{"containerPort":8081}]}],"serviceAccountName":"customers-service"}}}}
  creationTimestamp: "2026-01-04T05:18:55Z"
  generation: 2
  name: customers-service
  namespace: petclinic
  resourceVersion: "93151"
--
      serviceAccountName: customers-service
      terminationGracePeriodSeconds: 30
status:
  availableReplicas: 2
  conditions:
  - lastTransitionTime: "2026-01-04T07:01:38Z"


- kiem tra lai tat ca xem cho nao dang sai
- neu ko sai thi giai thich tai sao van ko chay dc va dua ra huong giai quyet tuong ung 