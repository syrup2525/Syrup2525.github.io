# Grafana & Prometheus 설치

## Namespace
``` bash
kubectl create ns prometheus
```

## 저장소 추가
``` bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

``` bash
helm repo update
```

## 설치
``` bash
helm install prometheus prometheus-community/kube-prometheus-stack -n prometheus
```

::: details `node-exporter` Pod 를 제외한 작업을 특정 노드에서 실행하기
`label` 및 `taint` 설정
``` bash
kubectl label node metric1 metric=true
kubectl taint node metric1 edicated=metric:NoSchedule
```

`values.yaml` 작성
::: code-group
``` yaml [values.yaml]
upgradeJob:
  nodeSelector:
    metric: "true"
  tolerations:
    - key: dedicated
      operator: Equal
      value: metric
      effect: NoSchedule

prometheus:
  prometheusSpec:
    nodeSelector:
      metric: "true"
    tolerations:
      - key: dedicated
        operator: Equal
        value: metric
        effect: NoSchedule

grafana:
  nodeSelector:
    metric: "true"
  tolerations:
    - key: dedicated
      operator: Equal
      value: metric
      effect: NoSchedule

alertmanager:
  alertmanagerSpec:
    nodeSelector:
      metric: "true"
    tolerations:
      - key: dedicated
        operator: Equal
        value: metric
        effect: NoSchedule

kube-state-metrics:
  nodeSelector:
    metric: "true"
  tolerations:
    - key: dedicated
      operator: Equal
      value: metric
      effect: NoSchedule

prometheusOperator:
  nodeSelector:
    metric: "true"
  tolerations:
    - key: dedicated
      operator: Equal
      value: metric
      effect: NoSchedule

  admissionWebhooks:
    patch:
      nodeSelector:
        metric: "true"
      tolerations:
        - key: dedicated
          operator: Equal
          value: metric
          effect: NoSchedule

# node-exporter만 전 노드 수집용으로 유지
prometheus-node-exporter:
  nodeSelector:
    kubernetes.io/os: linux
  tolerations:
    - operator: Exists
      effect: NoSchedule
```
:::

::: tip
#### k3s 환경에서 설치시 (안정버전)
``` bash
helm install prometheus prometheus-community/kube-prometheus-stack --version 70.4.2 -n prometheus
```

#### error converting YAML to JSON 오류 발생하는 경우
``` bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
DESIRED_VERSION="v3.17.3"
./get_helm.sh --version $DESIRED_VERSION
```
:::

## Issuer & Ingress
### Issuer
::: code-group
``` yaml [issuer.yaml]
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: prometheus
  namespace: prometheus
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

### Ingress
::: code-group
``` yaml [ingress.yaml]
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/issuer: prometheus
    cert-manager.io/issuer-kind: Issuer
  name: grafana
  namespace: prometheus
spec:
  ingressClassName: nginx
  rules:
    - host: grafana.example.com
      http:
        paths:
          - backend:
              service:
                name: prometheus-grafana
                port:
                  number: 80
            path: /
            pathType: Prefix
  tls:
    - hosts:
        - grafana.example.com
      secretName: tls-grafana-ingress
```
:::
``` bash
kubectl apply -f ingress.yaml
```

## Grafana WEB UI 접속
``` txt
https://grafana.example.com
```

::: tip
### 초기 아이디 & 비밀번호
* ID: admin
* PW: 
> ``` bash
> kubectl --namespace prometheus get secrets prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo
> ```
:::