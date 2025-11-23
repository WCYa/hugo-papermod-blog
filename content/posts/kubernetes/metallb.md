---
title: "🌐 Kubernetes 外部存取：MetalLB 裸機安裝與連線測試"
date: 2025-11-23T13:39:01+08:00
draft: false
tags: ["kubernetes", "metallb", "loadbalancer"]
---

## 目的

一般情況下要讓外部使用者使用 Kubernetes 的服務時，需要透過 External Load Balancers 如 Cloud Provider (GCP、AWS、Azure...)，但在使用 bare metal Kubernetes clusters 時，Kubernetes 沒有提供原生的這種 Network Load Balancers，如果要讓外部使用者能連線到 bare metal cluster 內部的服務時，通常是使用 NodePort 和 ExternalIPs service，但這兩種在生產環境使用都有著缺點。

MetalLB 的目的就是與標準網路設備做整合，使用標準的路由協定使在 bare-metal clusters 內的 `對外部的服務` 可以正常運作。

MetalLB 提供兩種模式：BGP mode 和 Layer 2 mode，BGP mode 才能實現真正的跨多節點均衡負載，但因為我只是測試用途所以採用 Layer 2 mode。

## MetalLB 在做什麼

MetalLB 會同時作到兩個功能去提供此服務：IP 位址分配、向外部宣告 IP 是存在於這個 cluster。

- IP 位址分配：你需要提供 IP Pools 給 MetalLB，其餘的事例如分配、取消分配、追蹤都由 MetalLB 處理。Private/Public 的 IPV4/IPV6 你要獨立提供或是一起提供都可以，取決於你的環境和租用的 IPs
- 向外部宣告 IP：向 cluster 所在的網路宣告，這個 IP 存在我這個 cluster，宣告方式取決於所使用的模式為何：ARP、NDP、or BGP

## 關於 Layer 2 模式

在 Layer 2 模式下，是一個節點負責向本地網路宣告某個 Service IP，故所有流到此 Service IP 的流量都先經過這台節點，之後才由 `kube-proxy` 去分散流量到此 Service 的所有 Pods。故 Layer 2 模式未實現真正的網路負載平衡，而是實現故障轉移的功能，故障轉移的偵測是使用 [memberlist](https://github.com/hashicorp/memberlist) 實現。

**那是誰負責去做宣告？**

只有一個節點去宣告某個 IP 是它負責，故需要為此 IP 選出誰是其 Leader，照官網的說法此選舉是無狀態的，工作原理如下：

- 每個 speakers 收集一個可能為某 IP 的宣告者的清單，會考慮幾個因素：active 的 speakers、external traffic policy、active 的 endpoints、node selectors 等等
- 每個 speakers 做相同的運算：使用 `node+VIP` 元素算出一個排序的哈希表，並且 Service IP 由此哈希表順序第一個的 item 所描述的節點 (speaker) 去負責宣告

**Layer 2 模式的限制**

主要有遇到兩個限制：單節點頻寬瓶頸、故障移轉可能延遲。

1. **頻寬限制：** 雖然 Layer 2 模式透過雜湊運算可將不同 Service IP 分散給不同節點，實現 Service 層級的負載均衡，但對於單一 Service (特別是聚合大流量的 Ingress Controller 和 Gateway API)，其所有流入流量仍受限於單一節點的頻寬與處理能力。
2. **故障移轉延遲：** MetalLB 透過發送 Gratuitous ARP 來宣告 IP 的新 MAC Address。雖然現代設備多能快速更新，但部份上層交換器或路由器的 ARP 快去策略可能較為保守 (或忽略GARP)，導致在 ARP Table 更新前，流量仍被送往故障節點，造成數秒至數分鐘的連線中斷。

## 安裝方式

### 準備

這邊我使用 kube-proxy 是啟用 IPVS mode，官網提到必須啟用 strict ARP mode：

```bash
# 查看此指令會更改什麼，如果有更動就會回傳非 0 值
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl diff -f - -n kube-system

# 實際去更新異動，如果錯誤會回傳非 0 值
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl apply -f - -n kube-system
```

### 透過 Helm 安裝 metallb

```bash
helm repo add metallb https://metallb.github.io/metallb

helm install metallb metallb/metallb --namespace metallb-system --create-namespace
```

此時 MetalLB 仍然是閒置狀態，直到設定了對應的 CRs (Custom Resources)。

### 設定 CRs

底下為範例 manifest 檔案，命名為 `metallb-l2-config.yaml`。因為我的環境是透過 [KubeSpray](https://github.com/kubernetes-sigs/kubespray/tree/master) 的 Vagrantfile 建立的，所以 IPAddressPool 就選用 172.18.8.0/24 網段。

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.18.8.201-172.18.8.205
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: first-ad
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
```

**建立 Resource Objects：**

```bash
kubectl apply -f metallb-l2-config.yaml
```

### 設定測試網頁

檔案名稱 `metallb-test.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-test-deployment
  labels:
    app: nginx-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-test
  template:
    metadata:
      labels:
        app: nginx-test
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-test-service
  # 如果需要指定取得 IPAddressPool 範圍內的特定 IP，加上以下 annotations
#  annotations:
#    metallb.io/loadBalancerIPs: 172.18.8.203
spec:
  selector:
    app: nginx-test
  ports:
    - protocol: TCP
      port: 80         # 外部連線用的 Port
      targetPort: 80   # Pod 內部的 Port
  type: LoadBalancer   # 類型設定為 LoadBalancer，MetalLB 會自動為這個 Service 分配 IP
```

**建立 Resource Objects：**

```bash
kubectl apply -f metallb-test.yaml
```

**檢查 Service 狀態和確認連線：**

```bash
kubectl get svc nginx-test-service
```

透過上述指令取得 EXTERNAL-IP 後，打開瀏覽器輸入 `http://{EXTERNAL-IP}` 應該能看到 "Welcome to nginx!" 的畫面

**移除測試網頁**

```bash
kubectl delete -f metallb-test.yaml
```
