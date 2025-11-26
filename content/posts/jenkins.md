---
title: "🤵 Jenkins：安裝 Jenkins 在 Kubernetes"
date: 2025-11-25T20:37:29+08:00
draft: false
tags: ["jenkins", "cicd", "kubernetes"]
---

## 事前準備

### 設定 Helm Repository

```bash
helm repo add jenkinsci https://charts.jenkins.io
helm repo update
# 可以使用以下指令查詢 Jenkins repo 存在的 charts
helm search repo jenkinsci
```

### 建立 Namespace

```bash
kubectl create namespace jenkins
```

### 建立持久性儲存 PV

1. 如果測試環境是使用 minikube，可直接參考 [官網範例](https://www.jenkins.io/doc/book/installing/kubernetes/#create-a-persistent-volume)
2. 建立 Local StorageClass，但需要能接收當節點不可被調度(如 CPU 使用率滿載)造成服務中斷跟資料遺失等等風險
3. 使用 Dynamic provisioner (例如：Rook)，則不用手動建立 PV

這邊使用 Local StorageClass 為例，這種方式有個好處，因為設定 `WaitForFirstConsumer` 時，可以延遲 Volume Binding 讓 scheduler 可以考慮所有 constrains，因此當 Local PV 建立好後，之後建立的 Pod 使用 Local StorageClass 時就會自動被調度到 PV 的節點上。

需要先決定 Local PV 要建立在哪個節點，這邊選用我的節點 `k8s-1`。

首先在節點 `k8s-1` 建立 Local PV 需要的目錄

```bash
mkdir -p /mnt/disks/ssd1
sudo chown 1000:1000 /mnt/disks/ssd1
```

以下為 StorageClass、PV 的 manifest 檔案，命名為 `jenkins-01-volume.yaml` ，如果節點名稱不一樣，需要調整 PV 的 `nodeAffinity` 的值：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner # indicates that this StorageClass does not support automatic provisioning
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-pv
spec:
  capacity:
    storage: 20Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - k8s-1
```

建立 Resource Objects：

```bash
kubectl apply -f jenkins-01-volume.yaml
```

### 建立 Service Account

- 檔名：`jenkins-02-sa.yaml`
- 官網連結：[jenkins-02-sa.yaml](https://raw.githubusercontent.com/jenkins-infra/jenkins.io/master/content/doc/tutorials/kubernetes/installing-jenkins-on-kubernetes/jenkins-02-sa.yaml)

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: jenkins
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: jenkins
rules:
- apiGroups:
  - '*'
  resources:
  - statefulsets
  - services
  - replicationcontrollers
  - replicasets
  - podtemplates
  - podsecuritypolicies
  - pods
  - pods/log
  - pods/exec
  - podpreset
  - poddisruptionbudget
  - persistentvolumes
  - persistentvolumeclaims
  - jobs
  - endpoints
  - deployments
  - deployments/scale
  - daemonsets
  - cronjobs
  - configmaps
  - namespaces
  - events
  - secrets
  verbs:
  - create
  - get
  - watch
  - delete
  - list
  - patch
  - update
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - get
  - list
  - watch
  - update
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: jenkins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:serviceaccounts:jenkins
```

建立 Resource Objects：

```bash
kubectl apply -f jenkins-02-sa.yaml
```

## 安裝 jenkins

### 調整 Helm Chart 參數

從官網下載 [jenkins-values.yaml](https://raw.githubusercontent.com/jenkinsci/helm-charts/main/charts/jenkins/values.yaml)，或者使用以下指令匯出成檔案：

```bash
helm show values jenkinsci/jenkins
```

需要調整的部份如下，：

- controller:
   - serviceType: LoadBalancer
      - 如果沒有 LoadBalancer 可以使用 NodePort，但我這邊使用 MetalLB，安裝方式可參考另一篇文章 『[Kubernetes 外部存取：MetalLB 裸機安裝與連線測試](https://wcya.github.io/hugo-papermod-blog/posts/kubernetes/metallb/)』
   - servicePort: 80
      - 如果使用 NodePort 就要調整為 30000–32767 其中一項，例如 32000
- persistence:
  - storageClass: local-storage
    - 使用上述建立的 StorageClass
- serviceAccount:
  - create: false
  - name: jenkins
    - 使用上述建立的 ServiceAccount 名稱
- rbac:
  - create: false
- loadBalancerSourceRanges:
  - 0.0.0.0/0 `(# 如果使用 MetalLB 將這行移除`，[問題](#使用-metallb-時遇到的問題))

### 安裝

```bash
helm install jenkins jenkinsci/jenkins -n jenkins -f jenkins-values.yaml
```

取的帳號 `admin` 的密碼：

```bash
kubectl get secret -n jenkins jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 -d ; echo
```

如果使用 LoadBalancer 使用以下指令取得 External IP

```bash
kubectl get service jenkins -n jenkins -o jsonpath="{.status.loadBalancer.ingress[0].ip}" ; echo
```

## 使用 MetalLB 時遇到的問題

當 LoadBalancer 使用 MetalLB 時，如果 jenkins chart 的 value.yaml 設定 `controller.loadBalancerSourceRanges: [ 0.0.0.0/0 ]` 時，發現 kube-proxy 在調整 iptables 時造成無法連線。

依照下面的資訊，流程如下：

1. 可以看到在 chain `KUBE-PROXY-FIREWALL` 因為符合了 ipset `KUBE-LOAD-BALANCER-FW` 內容而跳至 chain `KUBE-SOURCE-RANGES-FIREWALL`
2. 在 chain `KUBE-SOURCE-RANGES-FIREWALL` 第一條規則因為 ipset `KUBE-LOAD-BALANCER-SOURCE-CIDR` 為空所以不符合，繼續在同一條 chain 的下條規則
3. 在下條規則中，ipset `KUBE-LOAD-BALANCER-SOURCE-IP` 因為只有一條紀錄 `172.18.8.202,tcp:80,172.18.8.202`，但是因為這條的第三個欄位 src 是要等於 Service 本身的 IP `172.18.8.202`，所以外部的來源 IP 都無法符合，故此規則也不符合，繼續在同一條 chain 的下條規則
4. 最後一條規則是直接 DROP 掉封包，所以無法連線成功

kube-proxy 的 chain 如下：

```bash
-A KUBE-PROXY-FIREWALL -m comment --comment "Kubernetes service load balancer ip + port for load balancer with sourceRange" -m set --match-set KUBE-LOAD-BALANCER-FW dst,dst -j KUBE-SOURCE-RANGES-FIREWALL
-A KUBE-SOURCE-RANGES-FIREWALL -m comment --comment "Kubernetes service load balancer ip + port + source cidr for packet filter purpose" -m set --match-set KUBE-LOAD-BALANCER-SOURCE-CIDR dst,dst,src -j RETURN
-A KUBE-SOURCE-RANGES-FIREWALL -m comment --comment "Kubernetes service load balancer ip + port + source IP for packet filter purpose" -m set --match-set KUBE-LOAD-BALANCER-SOURCE-IP dst,dst,src -j RETURN
-A KUBE-SOURCE-RANGES-FIREWALL -j DROP
```

ipset 如下：

```bash
Name: KUBE-LOAD-BALANCER-FW
Type: hash:ip,port
Revision: 5
Header: family inet hashsize 1024 maxelem 65536
Size in memory: 256
References: 1
Number of entries: 1
Members:
172.18.8.202,tcp:80

Name: KUBE-LOAD-BALANCER-SOURCE-CIDR
Type: hash:ip,port,net
Revision: 7
Header: family inet hashsize 1024 maxelem 65536
Size in memory: 456
References: 1
Number of entries: 0
Members:

Name: KUBE-LOAD-BALANCER-SOURCE-IP
Type: hash:ip,port,ip
Revision: 5
Header: family inet hashsize 1024 maxelem 65536
Size in memory: 280
References: 1
Number of entries: 1
Members:
172.18.8.202,tcp:80,172.18.8.202
```

可以用以下指令去紀錄封包是否真的進入到 DROP 那條規則，但是因為 chain `KUBE-SOURCE-RANGES-FIREWALL` 是由 kube-proxy 管理，所以定時會被覆蓋，使用以下指令後要迅速使用瀏覽器或 curl 指令連線測試。

```bash
sudo iptables -I KUBE-SOURCE-RANGES-FIREWALL 3 -j LOG --log-prefix "KUBE_SOURCE_DROP_POINT: " --log-level 7 ; iptables-save | grep 'KUBE-SOURCE-RANGES-FIREWALL'
```

查看紀錄：

```bash
sudo dmesg | grep 'KUBE_SOURCE_DROP_POINT'
# 或者
sudo journalctl -k | grep 'KUBE_SOURCE_DROP_POINT'
```
