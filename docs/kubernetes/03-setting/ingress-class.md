# IngressClass
::: tip
`k3s` 설치시 기본 제공되는 `IngressClass` 가 아닌, `traefik` 을 사용한 신규 `IngressClass` 를 생성하는 방법을 설명합니다.
:::

## IngressClass 생성
### Namespace 생성
ingressClass 작업을 진행할 namespace 를 생성합니다. 이 예제에서는 `traefik-8443` 로 진행합니다.
``` bash
kubectl create namespace traefik-8443
```

### helm repo 추가
``` bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

### values.yaml 파일 작성
::: code-group
``` yaml [traefik-8443-values.yaml]
hostNetwork: true

deployment:
  dnsPolicy: ClusterFirstWithHostNet

ports:
  websecure:
    port: 8443          # 컨테이너 포트
    exposedPort: 8443   # (service 사용 시) 외부 포트
    hostPort: 8443      # 노드 포트 바인딩 - 핵심
    expose:
      default: true
    protocol: TCP
  metrics:
    port: 9101          # 별도 prometheus 구성을 위해 metrics 기본 포트 변경
    expose:
      default: false
    exposedPort: 9101
    protocol: TCP

additionalArguments:
  - "--entryPoints.websecure.address=:8443"
  - "--entryPoints.websecure.http.tls=true"

ingressClass:
  enabled: true
  isDefaultClass: false
  name: traefik-8443

service:
  enabled: false
```
:::

### helm chart 배포
``` bash
helm install traefik-8443 traefik/traefik \
  -n traefik-8443 \
  -f traefik-8443-values.yaml \
  --skip-crds
```

### 확인
#### Pod 확인
``` bash
kubectl get pods -n traefik-8443
```

#### Node (hostnetwork) 확인
``` bash
netstat -tnlp | grep 8443
```
