# Clearwater-IMS
- [k8s 安裝](#一kubernetes-安裝參考實驗室課程連結)
- [docker/containerd 設置](#二架設私有docker-registry)
- [IMS 部署](#三ims-部署)
- [SIP-stress測試](#四sip-stress測試)
- [附錄](#附錄)
- [重要！！相關工具](#相關工具)
  
## 一、Kubernetes 安裝參考實驗室課程[連結](https://github.com/CYCU-CDLAB/1142-k8s-document/blob/main/Lab0%20k8s%20installation/k8s_installation.md)

### 開始前於 master 確認：

```bash
kubectl get nodes                # 節點皆為 Ready
kubectl get po -n kube-system    # K8s 系統元件與 Calico 皆為 Running
docker version                   # docker 可用（build / push image 用）
```
---
## 二、架設私有Docker Registry

以下的 `{master IP}` 一律替換為 master 的實際 IP。

先於 **master** 設定 Docker 允許以 
HTTP 存取 registry（registry:2 預設走 HTTP，Docker 預設只信任 HTTPS）：

```bash
sudo tee /etc/docker/daemon.json <<EOF
{
  "insecure-registries": ["{master IP}:5000"]
}
EOF
sudo systemctl restart docker
```

> 重啟 docker 只影響 build/push；K8s 的 pod 由 containerd 管理，不受影響。

於 **master** 啟動 registry（資料存於 `/opt/registry`，開機自動啟動）：

```bash
docker run -d -p 5000:5000 --restart=always -v /opt/registry:/var/lib/registry --name registry registry:2
```

再於**所有節點**設定 containerd 信任這個 HTTP registry——kubelet 拉 image 走的是 containerd，需另外設定，與 Docker 的 `daemon.json` 互不相通：

```bash
sudo mkdir -p "/etc/containerd/certs.d/{master IP}:5000"
sudo tee "/etc/containerd/certs.d/{master IP}:5000/hosts.toml" <<EOF
server = "http://{master IP}:5000"

[host."http://{master IP}:5000"]
  capabilities = ["pull", "resolve"]
EOF

# 啟用 certs.d 設定目錄（預設 config_path 為空字串）
sudo sed -i "s|config_path = ''|config_path = '/etc/containerd/certs.d'|" /etc/containerd/config.toml
grep -n "config_path" /etc/containerd/config.toml    # 應顯示 /etc/containerd/certs.d
sudo systemctl restart containerd
```

> 重啟 containerd 不會中斷運行中的容器（容器由各自獨立的 shim 程序看管）。
---

## 三、IMS 部署
> [!NOTE]
> 這邊的檔案已經被我魔改成可以跑的，直接下載即可
> 
> 原始檔請參考 https://github.com/Metaswitch/clearwater-docker.git

### Download IMS file
```git clone https://github.com/jerry890926/Clearwater-IMS.git```

### Change directory
```cd Clearwater-IMS/clearwater-docker```

### 創建images

```bash
cd clearwater-docker
for i in base astaire cassandra chronos bono ellis homer homestead homestead-prov ralf sprout ; do docker build -t clearwater/$i $i ; done
```


### 將創建的images推送至Registry

```bash
for i in astaire cassandra chronos bono ellis homer homestead homestead-prov ralf sprout
do
    docker tag clearwater/$i:latest {master IP}:5000/clearwater/$i:latest
    docker push {master IP}:5000/clearwater/$i:latest
done
```

確認 registry 內容：

```bash
curl http://{master IP}:5000/v2/_catalog
# {"repositories":["clearwater/astaire","clearwater/bono", ... ]}
```

> [!NOTE]
> etcd 的原始載點 `quay.io/coreos/etcd:v2.2.5` 已從 quay.io 下架，改用備份在 Docker Hub 的副本（`jerry890926/ims-repo:etcd-v2.2.5`）。**樣板不改名，pod 仍使用原名 `quay.io/coreos/etcd:v2.2.5`**——做法是讓本地 registry 作為 quay.io 的 mirror。

於 **master** 把備份推進 registry：

```bash
docker pull jerry890926/ims-repo:etcd-v2.2.5
docker tag jerry890926/ims-repo:etcd-v2.2.5 {master IP}:5000/quay.io/coreos/etcd:v2.2.5
docker push {master IP}:5000/quay.io/coreos/etcd:v2.2.5
```

於**所有節點**新增 quay.io 的 mirror 設定——certs.d 的目錄名就是 image 名開頭的 registry host（`config_path` 已啟用；hosts.toml 於每次拉取時即時讀取，不需重啟 containerd）：

```bash
sudo mkdir -p /etc/containerd/certs.d/quay.io
sudo tee /etc/containerd/certs.d/quay.io/hosts.toml <<EOF
server = "https://quay.io"

[host."http://{master IP}:5000"]
  capabilities = ["pull", "resolve"]
EOF
```

驗證（任一節點，應成功拉下）：

```bash
sudo crictl pull quay.io/coreos/etcd:v2.2.5
```

### 創建configmap

```bash
kubectl create configmap env-vars --from-literal=ZONE=default.svc.cluster.local
```

### 部署工具 helm安裝

```bash
cd clearwater-docker/kubernetes
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```
### 執行 k8s-gencfg

```bash
./k8s-gencfg --image_path={master IP}:5000/clearwater --image_tag=latest
```

> [!NOTE]
> `k8s-gencfg` 會同時生成兩套配置yaml：
> - `kubernetes/` 頂層的yaml（給 `kubectl apply -f` 手動部署用）
> - `kubernetes/clearwater/` Helm chart。

>[!IMPORTANT]
> 要修改IMS的資源配置，請去修改 kubernetes/templates/*.tmpl 這些檔案
>
> 修改完後再次 ./k8s-gencfg ...，重新生成yaml檔 

### helm部署IMS系統

```bash
cd clearwater-docker/kubernetes

# 安裝
helm install clearwater clearwater
```

### 觀察clearwater IMS之Pod

```bash
kubectl get po
```

### 補充：卸載IMS

```bash
cd clearwater-docker/kubernetes

# 卸載
helm delete clearwater
```
---
## 四、SIP Stress測試

進入cassandra pod，註冊測試用的使用者：

```bash
/usr/share/clearwater/crest-prov/src/metaswitch/crest/tools/stress_provision.sh [numbers of subscribers]
```

進入ellis pod，下載sip-stress測試工具：

```bash
sudo su
apt install clearwater-sip-stress
cd /usr/share/clearwater/sip-stress
```

最後會看到兩個檔案，一個是測試用的使用者資訊（`users.csv.1`），另一個是測試用的場景（`sip-stress.xml`）。

把 `users.csv.1`、`scenario-revise.xml` 傳入 `/usr/share/clearwater/sip-stress` 路徑底下：

```bash
kubectl cp scenario-revise.xml ellis-c587b6c7d-5vnmx:/usr/share/clearwater/sip-stress
```

Testing：

```bash
/usr/share/clearwater/bin/sipp -i [ellis private IP] -sf ./scenario-revise.xml [Bono private IP]:5060 -t tn -s default.svc.cluster.local -inf ./users.csv.1 -r [call rate]
```

| 參數 | 說明 |
|---|---|
| `-i` | 執行traffic generator的instance IP |
| `-sf` | 測試場景的檔案位置 |
| `-t` | tn為TCP模式, un為UDP模式 |
| `-s` | IMS系統的domain |
| `-inf` | 使用者資訊的位置 |
| `-r` | 初始call rate |
| `-rate_increase` | 增加多少個call rate, default為1秒 |
| `-rate_max` | call rate到達多少時，停止測試 |
| `-fd [秒數]` | 和`-rate_increase`一起使用，設定每秒call rate增加多少 |
| `-trace_stat` | 輸出測試時的log檔 |
| `-trace_rtt` | 輸出Call的response time |


---
## 附錄
### Ksniff 封包捕獲

```bash
kubectl krew install sniff                  # 1. 安裝 Ksniff
apt-get install tshark                      # 2. 安裝 tshark
kubectl sniff -n [namespace] [pod name] -o - | tshark -r - -t ad   # 3. 捕獲封包
```

結果可寫成 `.pcap`，再以 Wireshark 解析，即可看到 Sprout 在 SIP 註冊與身分驗證流程中的角色。

---
## 相關工具
- IMS Bench SIPp — <https://sipp.sourceforge.net/ims_bench/reference.html#Installation>
- Jitsi — <https://jitsi.org/>
- Wireshark — <https://www.wireshark.org/download.html>
- Istio
  - <https://istio.io/latest/zh/docs/ops/deployment/deployment-models/>
  - <https://ithelp.ithome.com.tw/articles/10289718>
- 前人Istio的安裝教學
  - <https://hackmd.io/@willyttt/rybCIs7Co>
  - <https://hackmd.io/tw1nHriBSpaDq1I0OK1CJw>
- 前人OCM的安裝教學<https://hackmd.io/LYNoIKFVTzK6_nHLotGr_g>
- Jaeger(Istio觀測服務間的流量) - <https://ithelp.ithome.com.tw/m/articles/10253028>
