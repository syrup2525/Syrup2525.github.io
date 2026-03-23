# GitLab

## Namespace
``` bash
kubectl create ns gitlab
```
> gitlab 을 설치할 `gitlab` 를 정의 합니다.

## 저장소 추가
``` bash
helm repo add gitlab https://charts.gitlab.io
```

``` bash
helm repo update
```

## StorageClass 및 PV 생성
::: details Gitaly
#### StorageClass
gitaly-storage.yaml
``` yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: STORAGE_CLASS_NAME
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```
> * `STORAGE_CLASS_NAME` storage class 이름 `ex) gitaly-storage`
``` bash
kubectl apply -f gitaly-storage.yaml
```

#### PV
gitaly-pv.yaml
``` yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: PV_NAME // [!code warning]
spec:
  capacity:
    storage: 256Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: STORAGE_CLASS_NAME
  local:
    path: LOCAL_PATH // [!code warning]
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - NODE_NAME // [!code warning]
```
> * `STORAGE_CLASS_NAME` storage class 이름 `ex) gitaly-storage`
> * `PV_NAME` PV 의 이름
> * `LOCAL_PATH` path 경로 `ex) /mnt/path`
> * `NODE_NAME` NODE 이름 `ex) master1`
``` bash
kubectl apply -f gitaly-pv.yaml
```
:::

::: details Redis
#### StorageClass
redis-storage.yaml
``` yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: STORAGE_CLASS_NAME
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```
> * `STORAGE_CLASS_NAME` storage class 이름 `ex) redis-storage`
``` bash
kubectl apply -f redis-storage.yaml
```

#### PV
redis-pv.yaml
``` yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: PV_NAME // [!code warning]
spec:
  capacity:
    storage: 256Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: STORAGE_CLASS_NAME
  local:
    path: LOCAL_PATH // [!code warning]
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - NODE_NAME // [!code warning]
```
> * `STORAGE_CLASS_NAME` storage class 이름 `ex) redis-storage`
> * `PV_NAME` PV 의 이름
> * `LOCAL_PATH` path 경로 `ex) /mnt/path`
> * `NODE_NAME` NODE 이름 `ex) master1`
``` bash
kubectl apply -f redis-pv.yaml
```
:::

::: details MiniO
#### StorageClass
minio-storage.yaml
``` yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: STORAGE_CLASS_NAME
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```
> * `STORAGE_CLASS_NAME` storage class 이름 `ex) minio-storage`
``` bash
kubectl apply -f minio-storage.yaml
```

#### PV
minio-pv.yaml
``` yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: PV_NAME // [!code warning]
spec:
  capacity:
    storage: 256Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: STORAGE_CLASS_NAME
  local:
    path: LOCAL_PATH // [!code warning]
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - NODE_NAME // [!code warning]
```
> * `STORAGE_CLASS_NAME` storage class 이름 `ex) minio-storage`
> * `PV_NAME` PV 의 이름
> * `LOCAL_PATH` path 경로 `ex) /mnt/path`
> * `NODE_NAME` NODE 이름 `ex) master1`
``` bash
kubectl apply -f minio-pv.yaml
```
:::

::: details Postgresql
#### StorageClass
postgresql-storage.yaml
``` yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: STORAGE_CLASS_NAME
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```
> * `STORAGE_CLASS_NAME` storage class 이름 `ex) postgresql-storage`
``` bash
kubectl apply -f postgresql-storage.yaml
```

#### PV
postgresql-pv.yaml
``` yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: PV_NAME // [!code warning]
spec:
  capacity:
    storage: 256Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: STORAGE_CLASS_NAME
  local:
    path: LOCAL_PATH // [!code warning]
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - NODE_NAME // [!code warning]
```
> * `STORAGE_CLASS_NAME` storage class 이름 `ex) postgresql-storage`
> * `PV_NAME` PV 의 이름
> * `LOCAL_PATH` path 경로 `ex) /mnt/path`
> * `NODE_NAME` NODE 이름 `ex) master1`
``` bash
kubectl apply -f postgresql-pv.yaml
```
:::

## password 생성
gitlab root password 로 사용할 초기 비밀번호를 설정합니다

### base64 인코딩
``` bash
echo -n "password" | base64
```

::: code-group
``` yaml [secret.yaml]
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-root-password
  namespace: gitlab
data:
  password: cGFzc3dvcmQ= # base64 인코딩된 문자열 입력
```
:::

### secret 생성
``` bash
kubectl apply -f secret.yaml
```

## values.yaml 작성
::: code-group
``` yaml:line-numbers [values.yaml] {2,4,6,7,13,153,9,10,18,121,129,145,148,153,162,172}
global:
  edition: ce
  hosts:
    domain: example.com
  ingress:
    configureCertmanager: false
    enabled: false
  initialRootPassword:
    secret: gitlab-root-password
    key: password

certmanager:
  install: false

gitlab:
  gitaly:
    persistence:
      storageClass: CUSTOM_STORAGE_CLASS_NAME
      size: 50Gi
    resources:
      # We usually recommend not to specify default resources and to leave this as a conscious
      # choice for the user. This also increases chances charts run on environments with little
      # resources, such as Minikube. If you do want to specify resources, uncomment the following
      # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
      # limits:
      #  cpu: 100m
      #  memory: 128Mi
      requests:
        cpu: 100m
        memory: 200Mi
  gitlab-exporter:
    resources:
      # limits:
      #  cpu: 1
      #  memory: 2G
      requests:
        cpu: 75m
        memory: 100M
  gitlab-shell:
    resources:
      # We usually recommend not to specify default resources and to leave this as a conscious
      # choice for the user. This also increases chances charts run on environments with little
      # resources, such as Minikube. If you do want to specify resources, uncomment the following
      # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
      # limits:
      #  cpu: 100m
      #  memory: 128Mi
      requests:
        cpu: 0
        memory: 6M
    maxUnavailable: 1
    minReplicas: 2
    maxReplicas: 10
  kas:
    resources:
      requests:
        cpu: 100m
        memory: 100M
  mailroom:
    resources:
      # limits:
      #  cpu: 1
      #  memory: 2G
      requests:
        cpu: 50m
        memory: 150M
  migrations:
    resources:
      requests:
        cpu: 250m
        memory: 200Mi
  praefect:
    resources:
       requests:
        cpu: 100m
       memory: 200Mi
  sidekiq:
    resources:
      # limits:
      #  memory: 5G
      requests:
        cpu: 900m
        memory: 2G
  spamcheck:
    resources:
      requests:
        cpu: 100m
        memory: 100M
  toolbox:
    resources:
      # limits:
      #  cpu: 1
      #  memory: 2G
      requests:
        cpu: 50m
        memory: 350M
    backups:
      cron:
        requests:
          cpu: 50m
          memory: 350M
  webservice:
    resources:
      # limits:
      #  cpu: 1.5
      #  memory: 3G
      requests:
        cpu: 300m
        memory: 2.5G
    workhorse:
      resources:
        requests:
          cpu: 100m
          memory: 100M
    maxUnavailable: 1
    minReplicas: 2
    maxReplicas: 10

minio:
  persistence:
    storageClass: CUSTOM_STORAGE_CLASS_NAME
    size: 10Gi
  resources:
    requests:
      memory: 128Mi
      cpu: 100m

gitlab-runner:
  install: false

gitlab-zoekt:
  resources:
    # We usually recommend not to specify default resources and to leave this as a conscious
    # choice for the user. This also increases chances charts run on environments with little
    # resources, such as Minikube. If you do want to specify resources, uncomment the following
    # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
    # limits:
    #   cpu: 100m
    #   memory: 128Mi
    requests:
      cpu: 100m
      memory: 128Mi

nginx-ingress:
  enabled: false

prometheus:
  install: false

postgresql:
  primary:
    persistence:
      storageClass: CUSTOM_STORAGE_CLASS_NAME
    resources:
      limits: {}
      requests:
        memory: 256Mi
        cpu: 250m
  readReplicas:
    replicaCount: 1
    persistence:
      storageClass: CUSTOM_STORAGE_CLASS_NAME
    resources:
      limits: {}
      requests:
        memory: 256Mi
        cpu: 250m

redis:
  master:
    persistence:
      storageClass: CUSTOM_STORAGE_CLASS_NAME
      size: 5Gi
    resources:
      limits: {}
      requests: {}
  replica:
    replicaCount: 3
    resources:
      # We usually recommend not to specify default resources and to leave this as a conscious
      # choice for the user. This also increases chances charts run on environments with little
      # resources, such as Minikube. If you do want to specify resources, uncomment the following
      # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
      limits: {}
      #   cpu: 250m
      #   memory: 256Mi
      requests: {}
      #   cpu: 250m
      #   memory: 256Mi

registry:
  # define some sane resource requests and limitations
  resources:
    # limits:
    #   cpu: 200m
    #   memory: 1024Mi
    requests:
      cpu: 50m
      memory: 32Mi
```
:::

::: tip
* k8s 에서 `IngressClass` 를 1개만 사용하고자 하는 경우 `기존 IngressClass 를 사용하는 방법`을 진행합니다. (IngressClass 를 하나만 사용하지만 기존 IngressClass 를 연결하는 작업이 필요함)
* 기존에 생성된 `IngressClass` 를 사용하지 않고 자동 생성되는 `gitlab-ingress` 를 사용하고자 하는 경우 `gitlab 에서 제공하는 신규 IngressClass 를 사용하는 방법`을 진행합니다. (IngressClass 가 추가로 생성되지만 기존 IngressClass 를 연결하는 작업이 필요없음)
:::

::: details 기존 IngressClass 를 사용하는 방법
> * `2 line` ee > ce 변경 (enterprise edition > community edition)
> * `4 line` 현재 도메인 지정 
> * `6, 7, 13, 145 line` ingress class 를 자동 생성이 아닌 직접 등록을 위해 false 지정
> * `9, 10 line` 이전 단계에서 생성한 password secret 지정
> * `18 line` `gitaly` 에 대한 storageClass 지정 
> * `121 line` `minio` 에 대한 storageClass 지정 
> * `129 line` 추후 `gitlab-runner` 설치를 위해 `false` 설정
> * `148 line` 추후 별도로 `prometheus` 설치 예정이므로 `false` 설정
> * `153, 162 line` `postgresql` 에 대한 storageClass 지정 
> * `172 line` `redis` 에 대한 storageClass 지정 

gitlab helm chart 설치
``` bash
helm install gitlab gitlab/gitlab -f values.yaml -n gitlab
```
:::

::: details gitlab 에서 제공하는 신규 IngressClass 를 사용하는 방법
> * `2 line` ee > cc 변경 (enterprise edition > community edition)
> * `4 line` 현재 도메인 지정 
> * `6, 7, 145 line` ingress class 자동 생성을 위해 `true` 설정
> * `13 line` k8s 에 `certmanager` 가 설치되어 있는경우 `false`, 없는 경우 `true`
> * `9, 10 line` 이전 단계에서 생성한 password secret 지정
> * `18 line` `gitaly` 에 대한 storageClass 지정 
> * `121 line` `minio` 에 대한 storageClass 지정 
> * `129 line` 추후 `gitlab-runner` 설치를 위해 `false` 설정
> * `148 line` 추후 별도로 `prometheus` 설치 예정이므로 `false` 설정
> * `153, 162 line` `postgresql` 에 대한 storageClass 지정 
> * `172 line` `redis` 에 대한 storageClass 지정 

gitlab helm chart 설치
``` bash
helm install gitlab gitlab/gitlab -f values.yaml -n gitlab
```
:::

> resources 는 각 chart 에서 제공하는 기본값이며 필요시 튜닝하여 사용하면 됩니다.

## Ingress 생성
::: warning
* `values.yaml 작성` 단계에서 `기존 IngressClass 를 사용하는 방법` 으로 세팅한 경우 진행합니다.
* `values.yaml 작성` 단계에서 `gitlab 에서 제공하는 신규 IngressClass 를 사용하는 방법` 으로 진행 한 경우 `gitlab` 서비스를 위한 새로운 `IngressClass` 및 `Issuer`, `Ingress` 가 자동 생성되므로 추가적으로 생성할 필요가 없습니다.
:::
### Issuer 생성
::: code-group
``` yaml [issuer.yaml]
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: gitlab
  namespace: gitlab
spec:
  acme:
    email: example@email.com
    privateKeySecretRef:
      name: letsencrypt-production
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
      - http01:
          ingress:
            class: nginx
```
:::

``` bash
kubectl apply -f issuer.yaml
```

### GitLab
::: code-group
``` yaml [ingress.yaml]
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/issuer: gitlab
    cert-manager.io/issuer-kind: Issuer
    nginx.ingress.kubernetes.io/client-max-body-size: 1024m
    nginx.ingress.kubernetes.io/proxy-body-size: 1024m
  name: gitlab-webservice-default
  namespace: gitlab
spec:
  ingressClassName: nginx
  rules:
    - host: gitlab.example.com
      http:
        paths:
          - backend:
              service:
                name: gitlab-webservice-default
                port:
                  number: 8181
            path: /
            pathType: Prefix
  tls:
    - hosts:
        - gitlab.example.com
      secretName: tls-gitlab-ingress
```
:::

::: tip
`k3s` 를 통한 Ingress Class 를 `Traefik` 을 사용하는 경우 `nginx.ingress.kubernetes.io/client-max-body-size: 1024m`, `nginx.ingress.kubernetes.io/proxy-body-size: 1024m` 옵션 사용 대신 [timeout 시간 수정](/kubernetes/01-install/01-k3s/setting/traefik.html#timeout-시간-수정) 적용이 필요합니다
:::

``` bash
kubectl apply -f ingress.yaml
```

### Registry
::: code-group
``` yaml [ingress.yaml]
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/issuer: gitlab
    cert-manager.io/issuer-kind: Issuer
    nginx.ingress.kubernetes.io/client-max-body-size: 1024m
    nginx.ingress.kubernetes.io/proxy-body-size: 1024m
  name: gitlab-registry
  namespace: gitlab
spec:
  ingressClassName: nginx
  rules:
    - host: registry.example.com
      http:
        paths:
          - backend:
              service:
                name: gitlab-registry
                port:
                  number: 5000
            path: /
            pathType: Prefix
  tls:
    - hosts:
        - registry.example.com
      secretName: tls-registry-gitlab-ingress
```
:::
``` bash
kubectl apply -f ingress.yaml
```

### kas
::: code-group
``` yaml [ingress.yaml]
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/issuer: gitlab
    cert-manager.io/issuer-kind: Issuer
  name: gitlab-kas
  namespace: gitlab
spec:
  ingressClassName: nginx
  rules:
    - host: kas.example.com
      http:
        paths:
          - backend:
              service:
                name: gitlab-kas
                port:
                  number: 8154
            path: /k8s-proxy/
            pathType: Prefix
          - backend:
              service:
                name: gitlab-kas
                port:
                  number: 8150
            path: /
            pathType: Prefix
  tls:
    - hosts:
        - kas.example.com
      secretName: tls-kas-gitlab-ingress
```
:::
``` bash
kubectl apply -f ingress.yaml
```

### MiniO
::: code-group
``` yaml [ingress.yaml]
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/issuer: gitlab
    cert-manager.io/issuer-kind: Issuer
  name: gitlab-minio
  namespace: gitlab
spec:
  ingressClassName: nginx
  rules:
    - host: minio.example.com
      http:
        paths:
          - backend:
              service:
                name: gitlab-minio-svc
                port:
                  number: 9000
            path: /
            pathType: Prefix
  tls:
    - hosts:
        - minio.example.com
      secretName: tls-minio-gitlab-ingress
```
:::
``` bash
kubectl apply -f ingress.yaml
```

## 기타
### registry 정리 방법
``` bash
NS=gitlab
REL=gitlab

# 1. values 백업
helm get values -n "$NS" "$REL" -a > "${REL}-values.yaml"

# 2. read-only 전환
helm upgrade -n "$NS" "$REL" gitlab/gitlab \
  -f "${REL}-values.yaml" \
  --set registry.maintenance.readonly.enabled=true \
  --wait

# 3. registry pod 찾기
POD=$(kubectl get pods -n "$NS" -l app=registry -o jsonpath='{.items[0].metadata.name}')
echo "$POD"

# 4. 우선 안전하게 GC 실행(-m 없이)
kubectl exec -n "$NS" "$POD" -- /bin/registry garbage-collect /etc/docker/registry/config.yml

# 5. read-only 해제
helm upgrade -n "$NS" "$REL" gitlab/gitlab \
  -f "${REL}-values.yaml" \
  --wait
```