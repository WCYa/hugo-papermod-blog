---
title: "🌉 Gateway API：使用 Envoy Gateway"
date: 2025-12-21T17:03:11+08:00
draft: false
tags: ["kubernetes", "gatewayapi", "envoy"]
---

前幾篇文章使用了 cert-manager 產生 Private CA，並且安裝 MetalLB 讓 Kubernetes Cluster Service 可以被外部流量訪問，現在來試試看透過 Private CA 簽一個 Wildcard 憑證，並且安裝 Envoy Gateway 設定 SSL Termination，讓外部連線到 Jenkins 的流量可以被加密。

- [Kubernetes 中安裝 cert-manager 並產生 Private CA](https://wcya.github.io/hugo-papermod-blog/posts/kubernetes/cert-manager/)
- [Kubernetes 外部存取：MetalLB 裸機安裝與連線測試](https://wcya.github.io/hugo-papermod-blog/posts/kubernetes/metallb/)
- [Jenkins：安裝 Jenkins 在 Kubernetes](https://wcya.github.io/hugo-papermod-blog/posts/jenkins/install/)

## 安裝 Envoy Gateway

安裝：

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.6.1 -n envoy-gateway-system --create-namespace
```

確認服務處於有效狀態：

```bash
kubectl wait --timeout=5m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available
```

## 安裝 GatewayClass、Gateway

1. 首先建立一個 namespace 名為 `gateway`
2. Wildcard Certificate 使用先前產生的 Cluster Issuer `my-ca-issuer` 去簽，並且放到 `gateway` namespace
3. Gateway 設定 Wildcard Certificate 並且允許所有 namespace 都可以在其掛 Routes，此 Obejct 同樣放到 `gateway` namespace
4. 使用指令 `kubectl get gateway -n gateway eg`，查看 Gateway 取得的 External IP 是多少

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gateway
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-k8s-wcya-test
  namespace: gateway
spec:
  secretName: wildcard-k8s-wcya-test-tls
  issuerRef:
    name: my-ca-issuer
    kind: ClusterIssuer
  commonName: "*.k8s.wcya.test"
  dnsNames:
  - "*.k8s.wcya.test"
---
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: eg
  namespace: gateway
spec:
  gatewayClassName: eg
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
        - kind: Secret
          group: ""
          name: wildcard-k8s-wcya-test-tls
    allowedRoutes:
      namespaces:
        from: All
```

## 安裝 HTTPRoute

為 Jenkins Service 建立 HTTPRoute，並且 Gateway 設定剛剛建立的 `eg`

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: jenkins
  namespace: jenkins
spec:
  parentRefs:
  - name: eg
    namespace: gateway
  hostnames:
  - "jenkins.k8s.wcya.test"
  rules:
  - backendRefs:
    - name: jenkins
      port: 80
    matches:
    - path:
        type: PathPrefix
        value: /
```

## 調整域名解析

因為本地測試用，直接調整 `/etc/hosts`，Gateway `eg` 在我的環境是取得 `172.18.8.203`

```
172.18.8.203 jenkins.k8s.wcya.test
```

## Private CA 匯入信任憑證

以下是將 Private CA 匯入 Ubuntu，瀏覽器的部份依照不同瀏覽器則有不同的匯入方式。

從 Secret 匯出 Private CA，另存成 `my-selfsigned-ca.crt`：

```bash
kubectl -n cert-manager get secret root-secret -o jsonpath="{.data.ca\.crt}" | base64 -d
```

將憑證檔案放置目錄 `/usr/local/share/ca-certificates/`，並更新

```bash
sudo cp my-selfsigned-ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

查看是否匯入成功

```bash
openssl crl2pkcs7 -nocrl \
  -certfile /etc/ssl/certs/ca-certificates.crt \
  | openssl pkcs7 -print_certs -noout
```
