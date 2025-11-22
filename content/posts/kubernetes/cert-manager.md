---
title: "🔐 Kubernetes 中安裝 cert-manager 並產生 Private CA"
date: 2025-11-22T20:34:08+08:00
draft: false
tags: ["kubernetes", "cert-manager", "ca"]
---

> ⚠️ 僅用於測試用途

## 安裝

透過 Helm 安裝 cert-manager，首先下載簽署 cert-manager chart 的 GPG keyring，且在安裝時用此 GPG keyring 來驗證我們從 OCI Registry 下載的 chart

```bash
curl -LO https://cert-manager.io/public-keys/cert-manager-keyring-2021-09-20-1020CF3C033D4F35BAE1C19E1226061C665DF13E.gpg

helm install \
  cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --version v1.19.1 \
  --namespace cert-manager \
  --create-namespace \
  --verify \
  --keyring ./cert-manager-keyring-2021-09-20-1020CF3C033D4F35BAE1C19E1226061C665DF13E.gpg \
  --set crds.enabled=true
```

## 產生 Private CA 對應的 Custom Resources

以下為範例的 manifest 檔案，命名為 `root_ca.yaml`

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-selfsigned-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: my-selfsigned-ca
  secretName: root-secret
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
    group: cert-manager.io
  subject:
    organizations:
    - WCYa Example Inc.
    countries:
    - TW
    organizationalUnits:
    - Internal PKI
    localities:
    - Taipei
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: my-ca-issuer
spec:
  ca:
    # `ClusterIssuer` resource is not namespaced, so `secretName` is assumed to reference secret in `cert-manager` namespace.
    secretName: root-secret
```

**這些 Custom Resources 的用途：**

- `selfsigned-issuer` (ClusterIssuer)：用來引導 cert-manager 要使用本地的 PKI，讓產生 `my-selfsigned-ca` (Certificate) 時參照這個 `Issuer` 表示要自行簽署憑證
- `my-selfsigned-ca` (Certificate)：產生 Private CA，keypair 會被存在 Secret `root-secret`
  - 如果 `spec.subject` 未填寫，產生的憑證就不會包含 subject DN，從標準上來看此憑證是無效的，但測試用途的話可不必填寫，更準確的描述可參考官網 [Certificate Validity](https://cert-manager.io/docs/configuration/selfsigned/#certificate-validity)
- `my-ca-issuer` (ClusterIssuer)：後續用來簽署 Leaf Certificate (應用程式或使用者實際使用的憑證) 的 `Issuer`，簽署憑證時使用的是上述 `my-selfsigned-ca` 產生的 Private CA

**建立 Resource Objects：**

```bash
kubectl apply -f root_ca.yaml
```

**從 Secret 查看 CA 憑證：**

```bash
kubectl -n cert-manager get secret root-secret -o jsonpath="{.data.ca\.crt}" | base64 -d | openssl x509 -noout -text
```
