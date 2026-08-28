# Clearwater-IMS
- [k8s 安裝](1)
- [docker/containerd 設置](2)
- [IMS 部署](3)
  
[1] ## 一、Kubernetes 安裝參考實驗室課程[連結](https://github.com/CYCU-CDLAB/1142-k8s-document/blob/main/Lab0%20k8s%20installation/k8s_installation.md)

### 開始前於 master 確認：

```bash
kubectl get nodes                # 節點皆為 Ready
kubectl get po -n kube-system    # K8s 系統元件與 Calico 皆為 Running
docker version                   # docker 可用（build / push image 用）
```
---
[2] ## 二、架設私有Docker Registry

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

[3] ## 三、IMS 部署
### Download IMS file
```git clone https://github.com/jerry890926/Clearwater-IMS.git```

### Change directory
```cd Clearwater-IMS/clearwater-docker/kubernetes```
