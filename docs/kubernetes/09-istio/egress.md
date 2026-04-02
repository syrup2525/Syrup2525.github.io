# Istio 를 활용하여 Egress Node 구축

## Node 설정
``` bash
kubectl label node edge1 edge=true
kubectl label node edge2 edge=true
kubectl label node edge3 edge=true
kubectl label node edge4 edge=true

kubectl taint node edge1 dedicated=edge:NoSchedule
kubectl taint node edge2 dedicated=edge:NoSchedule
kubectl taint node edge3 dedicated=edge:NoSchedule
kubectl taint node edge4 dedicated=edge:NoSchedule
```

## istio 구축
### istio helm chart 
``` bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

### base 설치
``` bash
helm install istio-base istio/base -n istio-system --set defaultRevision=default --create-namespace
```

### Istio CNI 설치
``` bash
helm install istio-cni istio/cni -n istio-system --wait
```

### control plane 설치
``` bash
helm install istiod istio/istiod -n istio-system \
  --set pilot.cni.enabled=true \
  --set meshConfig.accessLogFile=/dev/stdout \
  --set meshConfig.accessLogEncoding=TEXT \
  --wait
```

### Egress 용 Gateway 구축
#### namespace 생성
``` bash
kubectl create namespace istio-egress
kubectl get ns istio-egress --show-labels
```

::: tip
labels 확인 단계에서, `istio-injection=disabled` 옵션이 없는지 확인
:::

#### egress-gateway-values.yaml 작성
::: code-group
```yaml [egress-gateway-values.yaml]
service:
  type: ClusterIP

nodeSelector:
  edge: "true"

tolerations:
  - key: dedicated
    operator: Equal
    value: edge
    effect: NoSchedule

autoscaling:
  enabled: false

replicaCount: 4
```
:::

#### gateway pod 배포
``` bash
helm install istio-egressgateway istio/gateway \
  -n istio-egress \
  -f egress-gateway-values.yaml \
  --wait
```

#### 배포 상태 확인
``` bash
kubectl get pods -n istio-egress -o wide
```

## 적용 확인
``` bash
kubectl create namespace istio-test
kubectl label namespace istio-test istio-injection=enabled
kubectl get ns istio-test --show-labels
```

### egress용 Gateway 리소스 만들기
::: tip
#### Gateway 역할
- “어느 문으로 받을지”만 정하는 설정
- Gateway는 메시 가장자리에서 동작하는 로드밸런서를 설명하며, 어떤 포트를 열고, 어떤 프로토콜을 쓰고, TLS/SNI를 어떻게 받을지 같은 걸 정의
- Gateway는 “들어온 트래픽을 어디로 보낼지”까지는 결정하지 않음
  + 해당 설정은 VirtualService 가 시행함. Gateway는 주로 L4~L6 수준 설정이고, 실제 애플리케이션 레벨 라우팅은 VirtualService를 바인딩
- Ingress Gateway → 외부에서 들어오는 요청을 받는 문
- Egress Gateway → 외부로 나가는 요청이 지나가는 출구 문
- 즉 Gateway는 **“문을 여는 역할”**까지
- 문을 열어놨다고 해서 자동으로 길이 정해지는 건 아님
:::

::: code-group
```yaml [gateway.yaml]
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: egressgateway-for-httpbin
  namespace: istio-egress
spec:
  selector:
    istio: egressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - httpbin.org
```
:::

### ServiceEntry 만들기
::: tip
#### ServiceEntry 역할
Istio가 원래 자동으로 알 수 없는 서비스를 수동으로 등록
- Kubernetes Service인 my-api.default.svc.cluster.local
  + 이건 Kubernetes가 원래 알고 있어서 보통 ServiceEntry가 필요 없음
- 외부 API api.github.com
  + Istio 가 따로 등록해주지 않으면 직접 등록 필요
- 외부 DB 192.168.100.61
  + Kubernetes Service가 아니라서 직접 등록 필요

즉, ServiceEntry 는   
“이 주소/도메인은 우리 시스템이 통신해도 되는 대상으로 인식해줘”   
라고 Istio에게 알려주는 설정
:::

::: code-group
```yaml [serviceEntry.yaml]
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: httpbin-ext
  namespace: istio-test
spec:
  hosts:
  - httpbin.org
  location: MESH_EXTERNAL
  resolution: DNS
  ports:
  - number: 80
    name: http
    protocol: HTTP
```
:::

### VirtualService로 “mesh → egress gateway → 외부” 경로 만들기
::: tip
#### VirtualService 역할
실제 라우팅 역할을 수행함   
VirtualService는 트래픽 라우팅에 영향을 주는 설정, 대상 host에 대해 HTTP/TCP 등 여러 포트의 트래픽 속성을 정의. 또 gateways 필드에 mesh를 쓰면 메시 내부 sidecar들에 적용할 수 있고, 특정 gateway 이름을 쓰면 그 gateway에만 적용할 수 있음

<br>

쉽게 말하면 VirtualService는 이런 역할을 수행함
- 이 요청이 오면 어디로 보내지?
- 특정 path면 v2로 보내지?
- 특정 host면 egress gateway로 먼저 보내지?
- 내부 sidecar에 적용할지, gateway에 적용할지?

egress 기준 예시
- 앱 pod가 api.example.com 으로 나가려 함
- VirtualService가 보고,
- “이건 바로 밖으로 보내지 말고”
- “먼저 istio-egressgateway 서비스로 보냄”
- “그 다음 egress gateway가 외부로 내보냄”

이렇게 트래픽의 길을 정하는것을 VirtualService 가 역할을 수행
:::

#### service 이름 확인
``` bash
kubectl get svc -n istio-egress
```

#### virtualService.yaml 작성
::: code-group
```yaml [virtualService.yaml]
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: route-httpbin-via-egress
  namespace: istio-test
spec:
  hosts:
  - httpbin.org
  gateways:
  - mesh
  - istio-egress/egressgateway-for-httpbin
  http:
  - match:
    - gateways:
      - mesh
      port: 80
    route:
    - destination:
        host: istio-egressgateway.istio-egress.svc.cluster.local
        port:
          number: 80
  - match:
    - gateways:
      - istio-egress/egressgateway-for-httpbin
      port: 80
    route:
    - destination:
        host: httpbin.org
        port:
          number: 80
```
:::

### telemetry 리소스 생성
::: code-group
```yaml [telemetry.yaml]
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: mesh-logging-default
  namespace: istio-system
spec:
  accessLogging:
  - providers:
    - name: envoy
```
:::

### 테스트
#### 테스트용 pod 생성
``` bash
kubectl apply -n istio-test -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: sleep
spec:
  containers:
  - name: sleep
    image: curlimages/curl:8.7.1
    command: ["/bin/sh", "-c", "sleep 3650d"]
EOF
```

#### Log 출력 (별개의 터미널에서)
``` bash
kubectl logs -n istio-egress deployment/istio-egressgateway \
  -c istio-proxy \
  --all-pods=true \
  --prefix \
  -f
```

#### curl 텍스트
``` bash
kubectl exec -n istio-test sleep -c sleep -- curl -I http://httpbin.org/get
```