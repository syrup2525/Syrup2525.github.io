# Istio 를 활용하여 Egress Node 구축

## istio 구축
### Node 설정
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

### 적용 확인
``` bash
kubectl create namespace istio-test
kubectl label namespace istio-test istio-injection=enabled
kubectl get ns istio-test --show-labels
```

## https 기반 Egress 구축하기
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

## TCP/IP 기반 Egress 구축하기
::: tip
먼저, 내부 egress-gateway 포트 매핑이 필요합니다.   
이 포트들은 클러스터 내부에서 egress gateway가 받는 포트일 뿐이고, 외부 DB 포트와 같을 필요는 없습니다.   
외부 TCP 서비스가 같은 포트를 여러 개 쓰면 구분이 어려워질 수 있어서 IP별 ServiceEntry와 대상별 egress-gateway 내부 포트 분리가 가장 안전합니다.

``` txt
MongoDB
192.168.1.11:27017 -> 27114
192.168.1.12:27017 -> 27115
192.168.1.13:27017 -> 27116
192.168.1.21:27017 -> 27131
192.168.1.22:27017 -> 27132
192.168.1.23:27017 -> 27133

MySQL/MariaDB
192.168.1.31:3306  -> 33120
192.168.1.32:3306  -> 33121
192.168.1.41:3306  -> 33151
192.168.1.42:3306  -> 33152
192.168.1.51:3306  -> 33156
192.168.1.52:3306  -> 33157
192.168.1.53:3306  -> 33158

Redis
192.168.1.61:6379  -> 63717
192.168.1.62:6379  -> 63718
192.168.1.63:6379  -> 63719
192.168.1.71:6379  -> 63741
192.168.1.72:6379  -> 63742
192.168.1.73:6379  -> 63743

Oracle
192.168.1.81:1521  -> 15211
192.168.1.82:1521  -> 15212
192.168.1.81:1523  -> 15231
192.168.1.82:1523  -> 15232
192.168.1.81:1555  -> 15551
192.168.1.82:1555  -> 15552
```
위 DB 연결을 예시로 Egress 로 분리하여 구하는 방법을 성명합니다.
:::

### `istio-egressgateway` Service 포트 추가
``` bash
kubectl edit svc -n istio-egress istio-egressgateway
```
기존 spec.ports: 아래에 아래 포트 추가
``` yaml
- name: tcp-mongo-11
  port: 27114
  protocol: TCP
  targetPort: 27114
- name: tcp-mongo-12
  port: 27115
  protocol: TCP
  targetPort: 27115
- name: tcp-mongo-13
  port: 27116
  protocol: TCP
  targetPort: 27116
- name: tcp-mongo-21
  port: 27131
  protocol: TCP
  targetPort: 27131
- name: tcp-mongo-22
  port: 27132
  protocol: TCP
  targetPort: 27132
- name: tcp-mongo-23
  port: 27133
  protocol: TCP
  targetPort: 27133

- name: tcp-mysql-31
  port: 33120
  protocol: TCP
  targetPort: 33120
- name: tcp-mysql-32
  port: 33121
  protocol: TCP
  targetPort: 33121
- name: tcp-mysql-41
  port: 33151
  protocol: TCP
  targetPort: 33151
- name: tcp-mysql-42
  port: 33152
  protocol: TCP
  targetPort: 33152
- name: tcp-mysql-51
  port: 33156
  protocol: TCP
  targetPort: 33156
- name: tcp-mysql-52
  port: 33157
  protocol: TCP
  targetPort: 33157
- name: tcp-mysql-53
  port: 33158
  protocol: TCP
  targetPort: 33158

- name: tcp-redis-61
  port: 63717
  protocol: TCP
  targetPort: 63717
- name: tcp-redis-62
  port: 63718
  protocol: TCP
  targetPort: 63718
- name: tcp-redis-63
  port: 63719
  protocol: TCP
  targetPort: 63719
- name: tcp-redis-71
  port: 63741
  protocol: TCP
  targetPort: 63741
- name: tcp-redis-72
  port: 63742
  protocol: TCP
  targetPort: 63742
- name: tcp-redis-73
  port: 63743
  protocol: TCP
  targetPort: 63743

- name: tcp-oracle-81-1521
  port: 15211
  protocol: TCP
  targetPort: 15211
- name: tcp-oracle-82-1521
  port: 15212
  protocol: TCP
  targetPort: 15212
- name: tcp-oracle-81-1523
  port: 15231
  protocol: TCP
  targetPort: 15231
- name: tcp-oracle-82-1523
  port: 15232
  protocol: TCP
  targetPort: 15232
- name: tcp-oracle-81-1555
  port: 15551
  protocol: TCP
  targetPort: 15551
- name: tcp-oracle-82-1555
  port: 15552
  protocol: TCP
  targetPort: 15552
```

### Gateway
Gateway는 egress gateway Pod가 어떤 내부 포트로 어떤 외부 DB 트래픽을 받을지 선언하는 역할입니다. Gateway 자체는 문을 여는 역할이고, 실제로 어디로 보낼지는 VirtualService가 정합니다.
::: code-group
``` yaml [gateway.yaml]
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: db-egressgateway
  namespace: istio-egress
spec:
  selector:
    istio: egressgateway
  servers:
  - port: { number: 27114, name: tcp-mongo-11, protocol: TCP }
    hosts: [ "mongo-11.db-egress.local" ]
  - port: { number: 27115, name: tcp-mongo-12, protocol: TCP }
    hosts: [ "mongo-12.db-egress.local" ]
  - port: { number: 27116, name: tcp-mongo-13, protocol: TCP }
    hosts: [ "mongo-13.db-egress.local" ]
  - port: { number: 27131, name: tcp-mongo-21, protocol: TCP }
    hosts: [ "mongo-21.db-egress.local" ]
  - port: { number: 27132, name: tcp-mongo-22, protocol: TCP }
    hosts: [ "mongo-22.db-egress.local" ]
  - port: { number: 27133, name: tcp-mongo-23, protocol: TCP }
    hosts: [ "mongo-23.db-egress.local" ]

  - port: { number: 33120, name: tcp-mysql-31, protocol: TCP }
    hosts: [ "mysql-31.db-egress.local" ]
  - port: { number: 33121, name: tcp-mysql-32, protocol: TCP }
    hosts: [ "mysql-32.db-egress.local" ]
  - port: { number: 33151, name: tcp-mysql-41, protocol: TCP }
    hosts: [ "mysql-41.db-egress.local" ]
  - port: { number: 33152, name: tcp-mysql-42, protocol: TCP }
    hosts: [ "mysql-42.db-egress.local" ]
  - port: { number: 33156, name: tcp-mysql-51, protocol: TCP }
    hosts: [ "mysql-51.db-egress.local" ]
  - port: { number: 33157, name: tcp-mysql-52, protocol: TCP }
    hosts: [ "mysql-52.db-egress.local" ]
  - port: { number: 33158, name: tcp-mysql-53, protocol: TCP }
    hosts: [ "mysql-53.db-egress.local" ]

  - port: { number: 63717, name: tcp-redis-61, protocol: TCP }
    hosts: [ "redis-61.db-egress.local" ]
  - port: { number: 63718, name: tcp-redis-62, protocol: TCP }
    hosts: [ "redis-62.db-egress.local" ]
  - port: { number: 63719, name: tcp-redis-63, protocol: TCP }
    hosts: [ "redis-63.db-egress.local" ]
  - port: { number: 63741, name: tcp-redis-71, protocol: TCP }
    hosts: [ "redis-71.db-egress.local" ]
  - port: { number: 63742, name: tcp-redis-72, protocol: TCP }
    hosts: [ "redis-72.db-egress.local" ]
  - port: { number: 63743, name: tcp-redis-73, protocol: TCP }
    hosts: [ "redis-73.db-egress.local" ]

  - port: { number: 15211, name: tcp-oracle-81-1521, protocol: TCP }
    hosts: [ "oracle-81.db-egress.local" ]
  - port: { number: 15212, name: tcp-oracle-82-1521, protocol: TCP }
    hosts: [ "oracle-82.db-egress.local" ]
  - port: { number: 15231, name: tcp-oracle-81-1523, protocol: TCP }
    hosts: [ "oracle-81.db-egress.local" ]
  - port: { number: 15232, name: tcp-oracle-82-1523, protocol: TCP }
    hosts: [ "oracle-82.db-egress.local" ]
  - port: { number: 15551, name: tcp-oracle-81-1555, protocol: TCP }
    hosts: [ "oracle-81.db-egress.local" ]
  - port: { number: 15552, name: tcp-oracle-82-1555, protocol: TCP }
    hosts: [ "oracle-82.db-egress.local" ]
```
:::

### ServiceEntry
ServiceEntry는 Istio가 원래 모르는 외부 대상을 서비스 레지스트리에 추가하는 리소스입니다. 포트 프로토콜은 HTTP|HTTPS|GRPC|HTTP2|MONGO|TCP|TLS 등을 쓸 수 있는데, 이번엔 IP 직결 + 외부 DB 공통 패턴이어서 전부 TCP로 통일하여 설정합니다.
#### MongoDB / MySQL / Redis
::: code-group 
``` yaml [mongodb-mysql-redis-serviceEntry.yaml]
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: mongo-11
  namespace: istio-egress
spec:
  hosts: [ "mongo-11.db-egress.local" ]
  addresses: [ "192.168.1.11/32" ]
  location: MESH_EXTERNAL
  resolution: STATIC
  ports:
  - number: 27017
    name: tcp-mongo
    protocol: TCP
  endpoints:
  - address: 192.168.1.11
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: mongo-12
  namespace: istio-egress
spec:
  hosts: [ "mongo-12.db-egress.local" ]
  addresses: [ "192.168.1.12/32" ]
  location: MESH_EXTERNAL
  resolution: STATIC
  ports:
  - number: 27017
    name: tcp-mongo
    protocol: TCP
  endpoints:
  - address: 192.168.1.12
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: mongo-13
  namespace: istio-egress
spec:
  hosts: [ "mongo-13.db-egress.local" ]
  addresses: [ "192.168.1.13/32" ]
  location: MESH_EXTERNAL
  resolution: STATIC
  ports:
  - number: 27017
    name: tcp-mongo
    protocol: TCP
  endpoints:
  - address: 192.168.1.13
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: mongo-21
  namespace: istio-egress
spec:
  hosts: [ "mongo-21.db-egress.local" ]
  addresses: [ "192.168.1.21/32" ]
  location: MESH_EXTERNAL
  resolution: STATIC
  ports:
  - number: 27017
    name: tcp-mongo
    protocol: TCP
  endpoints:
  - address: 192.168.1.21
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: mongo-22
  namespace: istio-egress
spec:
  hosts: [ "mongo-22.db-egress.local" ]
  addresses: [ "192.168.1.22/32" ]
  location: MESH_EXTERNAL
  resolution: STATIC
  ports:
  - number: 27017
    name: tcp-mongo
    protocol: TCP
  endpoints:
  - address: 192.168.1.22
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: mongo-23
  namespace: istio-egress
spec:
  hosts: [ "mongo-23.db-egress.local" ]
  addresses: [ "192.168.1.23/32" ]
  location: MESH_EXTERNAL
  resolution: STATIC
  ports:
  - number: 27017
    name: tcp-mongo
    protocol: TCP
  endpoints:
  - address: 192.168.1.23
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: mysql-31
  namespace: istio-egress
spec:
  hosts: [ "mysql-31.db-egress.local" ]
  addresses: [ "192.168.1.31/32" ]
  location: MESH_EXTERNAL
  resolution: STATIC
  ports:
  - number: 3306
    name: tcp-mysql
    protocol: TCP
  endpoints:
  - address: 192.168.1.31
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: mysql-32
  namespace: istio-egress
spec:
  hosts: [ "mysql-32.db-egress.local" ]
  addresses: [ "192.168.1.32/32" ]
  location: MESH_EXTERNAL
  resolution: STATIC
  ports:
  - number: 3306
    name: tcp-mysql
    protocol: TCP
  endpoints:
  - address: 192.168.1.32
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: mysql-41
  namespace: istio-egress
spec:
  hosts: [ "mysql-41.db-egress.local" ]
  addresses: [ "192.168.1.41/32" ]
  location: MESH_EXTERNAL
  resolution: STATIC
  ports:
  - number: 3306
    name: tcp-mysql
    protocol: TCP
  endpoints:
  - address: 192.168.1.41
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: mysql-42
  namespace: istio-egress
spec:
  hosts: [ "mysql-42.db-egress.local" ]
  addresses: [ "192.168.1.42/32" ]
  location: MESH_EXTERNAL
  resolution: STATIC
  ports:
  - number: 3306
    name: tcp-mysql
    protocol: TCP
  endpoints:
  - address: 192.168.1.42
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: mysql-51
  namespace: istio-egress
spec:
  hosts: [ "mysql-51.db-egress.local" ]
  addresses: [ "192.168.1.51/32" ]
  location: MESH_EXTERNAL
  resolution: STATIC
  ports:
  - number: 3306
    name: tcp-mysql
    protocol: TCP
  endpoints:
  - address: 192.168.1.51
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: mysql-52
  namespace: istio-egress
spec:
  hosts: [ "mysql-52.db-egress.local" ]
  addresses: [ "192.168.1.52/32" ]
  location: MESH_EXTERNAL
  resolution: STATIC
  ports:
  - number: 3306
    name: tcp-mysql
    protocol: TCP
  endpoints:
  - address: 192.168.1.52
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: mysql-53
  namespace: istio-egress
spec:
  hosts: [ "mysql-53.db-egress.local" ]
  addresses: [ "192.168.1.53/32" ]
  location: MESH_EXTERNAL
  resolution: STATIC
  ports:
  - number: 3306
    name: tcp-mysql
    protocol: TCP
  endpoints:
  - address: 192.168.1.53
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: redis-61
  namespace: istio-egress
spec:
  hosts: [ "redis-61.db-egress.local" ]
  addresses: [ "192.168.1.61/32" ]
  location: MESH_EXTERNAL
  resolution: STATIC
  ports:
  - number: 6379
    name: tcp-redis
    protocol: TCP
  endpoints:
  - address: 192.168.1.61
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: redis-62
  namespace: istio-egress
spec:
  hosts: [ "redis-62.db-egress.local" ]
  addresses: [ "192.168.1.62/32" ]
  location: MESH_EXTERNAL
  resolution: STATIC
  ports:
  - number: 6379
    name: tcp-redis
    protocol: TCP
  endpoints:
  - address: 192.168.1.62
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: redis-63
  namespace: istio-egress
spec:
  hosts: [ "redis-63.db-egress.local" ]
  addresses: [ "192.168.1.63/32" ]
  location: MESH_EXTERNAL
  resolution: STATIC
  ports:
  - number: 6379
    name: tcp-redis
    protocol: TCP
  endpoints:
  - address: 192.168.1.63
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: redis-71
  namespace: istio-egress
spec:
  hosts: [ "redis-71.db-egress.local" ]
  addresses: [ "192.168.1.71/32" ]
  location: MESH_EXTERNAL
  resolution: STATIC
  ports:
  - number: 6379
    name: tcp-redis
    protocol: TCP
  endpoints:
  - address: 192.168.1.71
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: redis-72
  namespace: istio-egress
spec:
  hosts: [ "redis-72.db-egress.local" ]
  addresses: [ "192.168.1.72/32" ]
  location: MESH_EXTERNAL
  resolution: STATIC
  ports:
  - number: 6379
    name: tcp-redis
    protocol: TCP
  endpoints:
  - address: 192.168.1.72
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: redis-73
  namespace: istio-egress
spec:
  hosts: [ "redis-73.db-egress.local" ]
  addresses: [ "192.168.1.73/32" ]
  location: MESH_EXTERNAL
  resolution: STATIC
  ports:
  - number: 6379
    name: tcp-redis
    protocol: TCP
  endpoints:
  - address: 192.168.1.73
```
:::

#### Oracle
::: code-group
``` yaml [oracle-serviceEntry.yaml]
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: oracle-81
  namespace: istio-egress
spec:
  hosts: [ "oracle-81.db-egress.local" ]
  addresses: [ "192.168.1.81/32" ]
  location: MESH_EXTERNAL
  resolution: STATIC
  ports:
  - { number: 1521, name: tcp-oracle-1521, protocol: TCP }
  - { number: 1523, name: tcp-oracle-1523, protocol: TCP }
  - { number: 1555, name: tcp-oracle-1555, protocol: TCP }
  endpoints:
  - address: 192.168.1.81
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: oracle-82
  namespace: istio-egress
spec:
  hosts: [ "oracle-82.db-egress.local" ]
  addresses: [ "192.168.1.82/32" ]
  location: MESH_EXTERNAL
  resolution: STATIC
  ports:
  - { number: 1521, name: tcp-oracle-1521, protocol: TCP }
  - { number: 1523, name: tcp-oracle-1523, protocol: TCP }
  - { number: 1555, name: tcp-oracle-1555, protocol: TCP }
  endpoints:
  - address: 192.168.1.82
```
:::

### VirtualService
VirtualService는 실제 라우팅 규칙입니다. mesh 쪽 규칙은 앱 sidecar → egress gateway 구간이고, istio-egress/db-egressgateway 쪽 규칙은 egress gateway → 실제 DB 구간입니다. mesh와 특정 gateway 둘 다 같은 VirtualService에 담을 수 있습니다.
#### MongoDB
::: code-group
``` yaml [mongodb-virtualService.yaml]
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: mongo-through-egress
  namespace: istio-egress
spec:
  hosts:
  - mongo-11.db-egress.local
  - mongo-12.db-egress.local
  - mongo-13.db-egress.local
  - mongo-21.db-egress.local
  - mongo-22.db-egress.local
  - mongo-23.db-egress.local
  gateways:
  - mesh
  - istio-egress/db-egressgateway
  tcp:
  - match: [{ gateways: [mesh], destinationSubnets: ["192.168.1.11/32"], port: 27017 }]
    route: [{ destination: { host: istio-egressgateway.istio-egress.svc.cluster.local, port: { number: 27114 } } }]
  - match: [{ gateways: [mesh], destinationSubnets: ["192.168.1.12/32"], port: 27017 }]
    route: [{ destination: { host: istio-egressgateway.istio-egress.svc.cluster.local, port: { number: 27115 } } }]
  - match: [{ gateways: [mesh], destinationSubnets: ["192.168.1.13/32"], port: 27017 }]
    route: [{ destination: { host: istio-egressgateway.istio-egress.svc.cluster.local, port: { number: 27116 } } }]
  - match: [{ gateways: [mesh], destinationSubnets: ["192.168.1.21/32"], port: 27017 }]
    route: [{ destination: { host: istio-egressgateway.istio-egress.svc.cluster.local, port: { number: 27131 } } }]
  - match: [{ gateways: [mesh], destinationSubnets: ["192.168.1.22/32"], port: 27017 }]
    route: [{ destination: { host: istio-egressgateway.istio-egress.svc.cluster.local, port: { number: 27132 } } }]
  - match: [{ gateways: [mesh], destinationSubnets: ["192.168.1.23/32"], port: 27017 }]
    route: [{ destination: { host: istio-egressgateway.istio-egress.svc.cluster.local, port: { number: 27133 } } }]

  - match: [{ gateways: ["istio-egress/db-egressgateway"], port: 27114 }]
    route: [{ destination: { host: mongo-11.db-egress.local, port: { number: 27017 } } }]
  - match: [{ gateways: ["istio-egress/db-egressgateway"], port: 27115 }]
    route: [{ destination: { host: mongo-12.db-egress.local, port: { number: 27017 } } }]
  - match: [{ gateways: ["istio-egress/db-egressgateway"], port: 27116 }]
    route: [{ destination: { host: mongo-13.db-egress.local, port: { number: 27017 } } }]
  - match: [{ gateways: ["istio-egress/db-egressgateway"], port: 27131 }]
    route: [{ destination: { host: mongo-21.db-egress.local, port: { number: 27017 } } }]
  - match: [{ gateways: ["istio-egress/db-egressgateway"], port: 27132 }]
    route: [{ destination: { host: mongo-22.db-egress.local, port: { number: 27017 } } }]
  - match: [{ gateways: ["istio-egress/db-egressgateway"], port: 27133 }]
    route: [{ destination: { host: mongo-23.db-egress.local, port: { number: 27017 } } }]
```
:::

#### MySQL / MariaDB
::: code-group
``` yaml [mysql-mariadb-virtualService.yaml]
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: mysql-through-egress
  namespace: istio-egress
spec:
  hosts:
  - mysql-31.db-egress.local
  - mysql-32.db-egress.local
  - mysql-41.db-egress.local
  - mysql-42.db-egress.local
  - mysql-51.db-egress.local
  - mysql-52.db-egress.local
  - mysql-53.db-egress.local
  gateways:
  - mesh
  - istio-egress/db-egressgateway
  tcp:
  - match: [{ gateways: [mesh], destinationSubnets: ["192.168.1.31/32"], port: 3306 }]
    route: [{ destination: { host: istio-egressgateway.istio-egress.svc.cluster.local, port: { number: 33120 } } }]
  - match: [{ gateways: [mesh], destinationSubnets: ["192.168.1.32/32"], port: 3306 }]
    route: [{ destination: { host: istio-egressgateway.istio-egress.svc.cluster.local, port: { number: 33121 } } }]
  - match: [{ gateways: [mesh], destinationSubnets: ["192.168.1.41/32"], port: 3306 }]
    route: [{ destination: { host: istio-egressgateway.istio-egress.svc.cluster.local, port: { number: 33151 } } }]
  - match: [{ gateways: [mesh], destinationSubnets: ["192.168.1.42/32"], port: 3306 }]
    route: [{ destination: { host: istio-egressgateway.istio-egress.svc.cluster.local, port: { number: 33152 } } }]
  - match: [{ gateways: [mesh], destinationSubnets: ["192.168.1.51/32"], port: 3306 }]
    route: [{ destination: { host: istio-egressgateway.istio-egress.svc.cluster.local, port: { number: 33156 } } }]
  - match: [{ gateways: [mesh], destinationSubnets: ["192.168.1.52/32"], port: 3306 }]
    route: [{ destination: { host: istio-egressgateway.istio-egress.svc.cluster.local, port: { number: 33157 } } }]
  - match: [{ gateways: [mesh], destinationSubnets: ["192.168.1.53/32"], port: 3306 }]
    route: [{ destination: { host: istio-egressgateway.istio-egress.svc.cluster.local, port: { number: 33158 } } }]

  - match: [{ gateways: ["istio-egress/db-egressgateway"], port: 33120 }]
    route: [{ destination: { host: mysql-31.db-egress.local, port: { number: 3306 } } }]
  - match: [{ gateways: ["istio-egress/db-egressgateway"], port: 33121 }]
    route: [{ destination: { host: mysql-32.db-egress.local, port: { number: 3306 } } }]
  - match: [{ gateways: ["istio-egress/db-egressgateway"], port: 33151 }]
    route: [{ destination: { host: mysql-41.db-egress.local, port: { number: 3306 } } }]
  - match: [{ gateways: ["istio-egress/db-egressgateway"], port: 33152 }]
    route: [{ destination: { host: mysql-42.db-egress.local, port: { number: 3306 } } }]
  - match: [{ gateways: ["istio-egress/db-egressgateway"], port: 33156 }]
    route: [{ destination: { host: mysql-51.db-egress.local, port: { number: 3306 } } }]
  - match: [{ gateways: ["istio-egress/db-egressgateway"], port: 33157 }]
    route: [{ destination: { host: mysql-52.db-egress.local, port: { number: 3306 } } }]
  - match: [{ gateways: ["istio-egress/db-egressgateway"], port: 33158 }]
    route: [{ destination: { host: mysql-53.db-egress.local, port: { number: 3306 } } }]
```
:::

#### Redis
::: code-group
``` yaml [redis-virtualService.yaml]
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: redis-through-egress
  namespace: istio-egress
spec:
  hosts:
  - redis-61.db-egress.local
  - redis-62.db-egress.local
  - redis-63.db-egress.local
  - redis-71.db-egress.local
  - redis-72.db-egress.local
  - redis-73.db-egress.local
  gateways:
  - mesh
  - istio-egress/db-egressgateway
  tcp:
  - match: [{ gateways: [mesh], destinationSubnets: ["192.168.1.61/32"], port: 6379 }]
    route: [{ destination: { host: istio-egressgateway.istio-egress.svc.cluster.local, port: { number: 63717 } } }]
  - match: [{ gateways: [mesh], destinationSubnets: ["192.168.1.62/32"], port: 6379 }]
    route: [{ destination: { host: istio-egressgateway.istio-egress.svc.cluster.local, port: { number: 63718 } } }]
  - match: [{ gateways: [mesh], destinationSubnets: ["192.168.1.63/32"], port: 6379 }]
    route: [{ destination: { host: istio-egressgateway.istio-egress.svc.cluster.local, port: { number: 63719 } } }]
  - match: [{ gateways: [mesh], destinationSubnets: ["192.168.1.71/32"], port: 6379 }]
    route: [{ destination: { host: istio-egressgateway.istio-egress.svc.cluster.local, port: { number: 63741 } } }]
  - match: [{ gateways: [mesh], destinationSubnets: ["192.168.1.72/32"], port: 6379 }]
    route: [{ destination: { host: istio-egressgateway.istio-egress.svc.cluster.local, port: { number: 63742 } } }]
  - match: [{ gateways: [mesh], destinationSubnets: ["192.168.1.73/32"], port: 6379 }]
    route: [{ destination: { host: istio-egressgateway.istio-egress.svc.cluster.local, port: { number: 63743 } } }]

  - match: [{ gateways: ["istio-egress/db-egressgateway"], port: 63717 }]
    route: [{ destination: { host: redis-61.db-egress.local, port: { number: 6379 } } }]
  - match: [{ gateways: ["istio-egress/db-egressgateway"], port: 63718 }]
    route: [{ destination: { host: redis-62.db-egress.local, port: { number: 6379 } } }]
  - match: [{ gateways: ["istio-egress/db-egressgateway"], port: 63719 }]
    route: [{ destination: { host: redis-63.db-egress.local, port: { number: 6379 } } }]
  - match: [{ gateways: ["istio-egress/db-egressgateway"], port: 63741 }]
    route: [{ destination: { host: redis-71.db-egress.local, port: { number: 6379 } } }]
  - match: [{ gateways: ["istio-egress/db-egressgateway"], port: 63742 }]
    route: [{ destination: { host: redis-72.db-egress.local, port: { number: 6379 } } }]
  - match: [{ gateways: ["istio-egress/db-egressgateway"], port: 63743 }]
    route: [{ destination: { host: redis-73.db-egress.local, port: { number: 6379 } } }]
```
:::

#### Oracle
::: code-group
``` yaml [oracle-virtualService.yaml]
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: oracle-through-egress
  namespace: istio-egress
spec:
  hosts:
  - oracle-81.db-egress.local
  - oracle-82.db-egress.local
  gateways:
  - mesh
  - istio-egress/db-egressgateway
  tcp:
  - match: [{ gateways: [mesh], destinationSubnets: ["192.168.1.81/32"], port: 1521 }]
    route: [{ destination: { host: istio-egressgateway.istio-egress.svc.cluster.local, port: { number: 15211 } } }]
  - match: [{ gateways: [mesh], destinationSubnets: ["192.168.1.82/32"], port: 1521 }]
    route: [{ destination: { host: istio-egressgateway.istio-egress.svc.cluster.local, port: { number: 15212 } } }]
  - match: [{ gateways: [mesh], destinationSubnets: ["192.168.1.81/32"], port: 1523 }]
    route: [{ destination: { host: istio-egressgateway.istio-egress.svc.cluster.local, port: { number: 15231 } } }]
  - match: [{ gateways: [mesh], destinationSubnets: ["192.168.1.82/32"], port: 1523 }]
    route: [{ destination: { host: istio-egressgateway.istio-egress.svc.cluster.local, port: { number: 15232 } } }]
  - match: [{ gateways: [mesh], destinationSubnets: ["192.168.1.81/32"], port: 1555 }]
    route: [{ destination: { host: istio-egressgateway.istio-egress.svc.cluster.local, port: { number: 15551 } } }]
  - match: [{ gateways: [mesh], destinationSubnets: ["192.168.1.82/32"], port: 1555 }]
    route: [{ destination: { host: istio-egressgateway.istio-egress.svc.cluster.local, port: { number: 15552 } } }]

  - match: [{ gateways: ["istio-egress/db-egressgateway"], port: 15211 }]
    route: [{ destination: { host: oracle-81.db-egress.local, port: { number: 1521 } } }]
  - match: [{ gateways: ["istio-egress/db-egressgateway"], port: 15212 }]
    route: [{ destination: { host: oracle-82.db-egress.local, port: { number: 1521 } } }]
  - match: [{ gateways: ["istio-egress/db-egressgateway"], port: 15231 }]
    route: [{ destination: { host: oracle-81.db-egress.local, port: { number: 1523 } } }]
  - match: [{ gateways: ["istio-egress/db-egressgateway"], port: 15232 }]
    route: [{ destination: { host: oracle-82.db-egress.local, port: { number: 1523 } } }]
  - match: [{ gateways: ["istio-egress/db-egressgateway"], port: 15551 }]
    route: [{ destination: { host: oracle-81.db-egress.local, port: { number: 1555 } } }]
  - match: [{ gateways: ["istio-egress/db-egressgateway"], port: 15552 }]
    route: [{ destination: { host: oracle-82.db-egress.local, port: { number: 1555 } } }]
```
:::

### 테스트
``` bash
kubectl logs -n istio-egress deployment/istio-egressgateway \
  -c istio-proxy \
  --all-pods=true \
  --prefix \
  -f
```

다른 터미널에서

#### Redis
``` bash
kubectl exec -it -n <namespace-name> <redis-client-pod> -- redis-cli -h 192.168.1.61 -p 6379 -a '비밀번호' PING
```

#### MySQL / MariaDB
``` bash
kubectl exec -it -n <namespace-name> <mysql-client-pod> -- mysql -h 192.168.1.31 -P 3306 -u <user> -p
```

#### Oracle
``` bash
kubectl exec -it -n <namespace-name> <oracle-client-pod> -- nc -vz 192.168.1.81 1521
```

#### MongoDB
``` bash
kubectl exec -it -n <namespace-name>  <mongo-client-pod> -- mongosh --host 192.168.1.11 --port 27017
```