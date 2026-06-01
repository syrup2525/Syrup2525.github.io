# Reverse Proxy

::: tip
Ingress 를 이용하여 사설 IP `192.168.1.100` 로 Reverse Proxy 하는 방법을 설명합니다.
:::

## 구조
``` txt
Client
  ↓
Ingress Controller
  ↓
Ingress
  ↓
K8s Service
  ↓
EndpointSlice
  ↓
192.168.1.100:3000
```

## Service 생성
::: code-group
``` yaml [service.yaml]
apiVersion: v1
kind: Service
metadata:
  name: external-app
  namespace: default
spec:
  type: ClusterIP
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 3000
```
:::

## EndpointSlice 생성
::: code-group 
``` yaml [endpointslice.yaml]
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: external-app-1
  namespace: default
  labels:
    kubernetes.io/service-name: external-app
addressType: IPv4
ports:
  - name: http
    protocol: TCP
    port: 3000
endpoints:
  - addresses:
      - "192.168.1.100"
```
:::

## Ingress 생성
::: code-group 
``` yaml [ingress.yaml]
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: external-app-ingress
  namespace: default
spec:
  ingressClassName: nginx
  rules:
    - host: external-app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: external-app
                port:
                  number: 80
```
:::

## 적용
``` bash
kubectl apply -f service.yaml
kubectl apply -f endpointslice.yaml
kubectl apply -f ingress.yaml
```

