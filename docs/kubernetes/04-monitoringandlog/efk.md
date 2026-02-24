
# EFK

## Namespace
``` bash
kubectl create ns efk
```
> EFK Stack 을 설치할 `namespace` 를 정의 합니다.

## Elasticsearch
::: tip
[Elasticsearch GitHub](https://github.com/elastic/helm-charts/blob/main/elasticsearch/README.md)
:::

::: tip
`StorageClass` 및 `PV` 가 미리 생성 되어 있어야합니다.

::: details `StorageClass` 및 `PV` 생성
#### StorageClass
``` yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```
> `local-storage.yaml` `StorageClass` 파일 작성
``` bash
kubectl apply -f local-storage.yaml
```
> local-storage StorageClass 생성
#### PV
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
  storageClassName: local-storage
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
> `pv-master1.yaml` `PV` 생성

> * `PV_NAME` 예시 local-pv-master1 과 같은 PV 의 이름
> * `LOCAL_PATH` 예시 /mnt/path 와 같은 path 경로
> * `NODE_NAME` 예시 master1 과 같은 NODE 이름 입력

``` bash
kubectl apply -f pv-master1.yaml
kubectl apply -f pv-master2.yaml
...
```
> 노드 별로 PV 를 생성하여 각각 배포
:::

### 저장소 추가
``` bash
helm repo add elastic https://helm.elastic.co
```

``` bash
helm repo update
```

### 설치
::: tip
[Elasticsearch values.yaml 참고](https://github.com/elastic/helm-charts/blob/main/elasticsearch/values.yaml)
:::

::: code-group
``` yaml [values.yaml] {3-4,6-7,9-10,14,17}
resources:
  requests:
    cpu: "1000m"
    memory: "2Gi"
  limits:
    cpu: "1000m"
    memory: "2Gi"

replicas: 3
minimumMasterNodes: 2

volumeClaimTemplate:
  accessModes: ["ReadWriteOnce"]
  storageClassName: local-storage
  resources:
    requests:
      storage: 30Gi
```
:::
> values.yaml 파일을 현재 시스템 상황에 맞게 적절히 수정합니다.

::: details Elasticsearch 단일 노드 구성 시
::: code-group
``` yaml [values.yaml] {12}
resources:
  requests:
    cpu: "1000m"
    memory: "2Gi"
  limits:
    cpu: "1000m"
    memory: "2Gi"

replicas: 1
minimumMasterNodes: 1

clusterHealthCheckParams: "wait_for_status=yellow&timeout=5s"

volumeClaimTemplate:
  accessModes: ["ReadWriteOnce"]
  storageClassName: efk-storage
  resources:
    requests:
      storage: 30Gi
```
:::

``` bash
helm install elasticsearch -f values.yaml elastic/elasticsearch -n efk --version 8.5.1
```
> elasticsearch 설치

## Fluent-bit
### 저장소 추가
``` bash
helm repo add fluent https://fluent.github.io/helm-charts
```

``` bash
helm repo update
```

### 설치
* values.yaml 파일 작성
::: code-group
``` yaml:line-numbers [values.yaml] {12-14,25-27,37-44,53-62}
daemonSetVolumes:
  - name: varlog
    hostPath:
      path: /var/log
  - name: varlibdockercontainers
    hostPath:
      path: /var/lib/docker/containers
  - name: etcmachineid
    hostPath:
      path: /etc/machine-id
      type: File
  - name: ca-certificates
    secret:
      secretName: elasticsearch-master-certs

daemonSetVolumeMounts:
  - name: varlog
    mountPath: /var/log
  - name: varlibdockercontainers
    mountPath: /var/lib/docker/containers
    readOnly: true
  - name: etcmachineid
    mountPath: /etc/machine-id
    readOnly: true
  - name: ca-certificates
    mountPath: /fluent-bit/secrets/ca-certificates
    readOnly: true

config:
  outputs: |
    [OUTPUT]
        Name es
        Match kube.*
        Host elasticsearch-master
        Logstash_Format On
        Retry_Limit False
        Trace_Error On
        HTTP_User elastic
        HTTP_Passwd HTTP_PASSWORD
        tls On
        tls.verify On
        tls.ca_file /fluent-bit/secrets/ca-certificates/ca.crt
        Suppress_Type_Name On
        Replace_Dots On

    [OUTPUT]
        Name es
        Match host.*
        Host elasticsearch-master
        Logstash_Format On
        Logstash_Prefix node
        Retry_Limit False
        Trace_Error On
        HTTP_User elastic
        HTTP_Passwd HTTP_PASSWORD
        tls On
        tls.verify On
        tls.ca_file /fluent-bit/secrets/ca-certificates/ca.crt
        Suppress_Type_Name On
        Replace_Dots On
```
:::
::: tip
> `highlighting` 되지 않은 영역은 기존 [values.yaml](https://github.com/fluent/helm-charts/blob/main/charts/fluent-bit/values.yaml) 값이고 `highlighting` 된 영역은 추가로 작성한 설정값 입니다.

> [elasticsearch](https://docs.fluentbit.io/manual/pipeline/outputs/elasticsearch) 및 [TLS/SSL](https://docs.fluentbit.io/manual/administration/transport-security) 환경변수 공식문서 참고
:::

> * <b>`12-14 line`</b> elasticsearch tls 인 secret 을 지정합니다.
> * <b>`25-27 line`</b> tls 파일의 경로를 지정합니다.
> * <b>`37,53 line`</b> <b>`Trace_Error`</b> 오류 출력을 지정합니다. (기본값 Off)
> * <b>`38,54 line`</b> <b>`HTTP_User`</b> 별도로 변경하지 않았으면 `elastic` 입니다.
> * <b>`39,55 line`</b> <b>`HTTP_Passwd`</b> 아래 명령어로 비밀번호를 확인 후 입력합니다.
> ``` bash
> kubectl get secrets --namespace=efk elasticsearch-master-credentials -ojsonpath='{.data.password}' | base64 -d
> ```
> * <b>`40,56 line`</b> <b>`tls`</b> elasticsearch 7.x 부터 https 가 기본이므로 `On` 으로 설정합니다. (기본값 Off)
> * <b>`41,57 line`</b> <b>`tls.verify`</b> tls 인증을 설정합니다. (기본값 Off)
> * <b>`42,58 line`</b> <b>`tls.ca_file`</b> `26 line` 에서 지정한 `mountPath` 값을 입력합니다.
> * <b>`43,59 line`</b> <b>`Suppress_Type_Name`</b> `illegal_argument_exception` 오류 해결을 위해 `On` 으로 설정합니다. [stackoverflow](https://stackoverflow.com/questions/69617608/elasticsearch-8-errors-with-action-metadata-line-1-contains-an-unknown-paramet)
> * <b>`44,60 line`</b> <b>`Replace_Dots`</b> `mapper_parsing_exception` 오류 해결을 위해 `On` 으로 설정합니다. [stackoverflow](https://stackoverflow.com/questions/62975255/fluent-bit-cannot-parse-kubernetes-logs)

::: tip
`read error, check permissions: /var/log/containers/*.log` 에러가 발생하는 경우 `values.yaml` 에 다음을 추가합니다

``` yaml
securityContext:
  runAsUser: 0
  runAsGroup: 0
  privileged: true
```
:::

* helm install fluent-bit

``` bash
helm install fluent-bit fluent/fluent-bit -f values.yaml -n efk --version 0.47.10
```
> chart version `0.47.10` Application version `3.1.9`

## Kibana
::: tip
[Kibana GitHub](https://github.com/elastic/helm-charts/tree/main/kibana)
:::

### 설치
::: tip
[Elasticsearch values.yaml 참고](https://github.com/elastic/helm-charts/blob/main/kibana/values.yaml)
:::

::: code-group
``` yaml [values.yaml] 
resources:
  requests:
    cpu: "1000m"
    memory: "2Gi"
  limits:
    cpu: "1000m"
    memory: "2Gi"
```
:::
> values.yaml 파일을 현재 시스템 상황에 맞게 적절히 수정합니다.

``` bash
helm install kibana elastic/kibana -f values.yaml -n efk --version 8.5.1
```
> elasticsearch 설치

### Ingress 설정
#### Issuer 생성
::: code-group
``` yaml [issuer.yaml]
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: efk
  namespace: efk
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

#### Ingress 생성
::: code-group
``` yaml [ingress.yaml]
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/issuer: efk
    cert-manager.io/issuer-kind: Issuer
  name: kibana
  namespace: efk
spec:
  ingressClassName: nginx
  rules:
    - host: kibana.yourdomain.com
      http:
        paths:
          - backend:
              service:
                name: kibana-kibana
                port:
                  number: 5601
            path: /
            pathType: ImplementationSpecific
  tls:
    - hosts:
        - kibana.yourdomain.com
      secretName: tls-kibana-ingress
```
:::
> * `metadata.annotations.cert-manager.io/issuer` issuer.yaml 에서 작성한 metadata.name
> * `spec.tls[0].hosts` TLS 를 적용할 도메인 이름
> * `spec.tls[0].secretName` 생성될 Secret 의 이름

``` bash
kubectl apply -f ingress.yaml 
```
#### 접속
`https://kibana.yourdomain.com` 로 접속후 `Username` elastic `Password` 는 아래 명령어로 확인 후 로그인 합니다.

> ``` bash
> kubectl get secrets --namespace=efk elasticsearch-master-credentials -ojsonpath='{.data.password}' | base64 -d
> ```

#### 
1. 메인화면 2/3 지점 우측 하단 `⚙ Stack Management` 선택
2. 우측 네비게이션에서 `Kibana` > `Data Views` 선택
3. 화면 중간 `Create data view` 선택
4. `Name` 에 fluent-bit `Index pettern` 에 logstash-* 입력
5. `Save data view to Kibana` 선택