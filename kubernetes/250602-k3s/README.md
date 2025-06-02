# K3s 初探

## 简介

K3s 是由 Rancher Labs 开发的轻量级 Kubernetes 发行版，专为资源受限的环境设计。它是一个完全兼容的 Kubernetes 发行版，但具有以下特点：

- **轻量级**：二进制文件小于 100MB
- **简单**：单一二进制文件，易于安装和维护
- **安全**：默认启用 TLS 和 RBAC
- **低资源消耗**：内存占用比标准 Kubernetes 减少 50%
- **快速启动**：几秒钟内启动完整的 Kubernetes 集群
- **自包含**：包含所有必需的依赖项
- **高可用性**：支持嵌入式 etcd 和外部数据库

## 内置组件
- **容器运行时**：containerd
- **网络插件**：Flannel (默认)
- **存储类**：Local Path Provisioner
- **负载均衡器**：Klipper Load Balancer
- **Ingress 控制器**：Traefik (默认)
- **DNS**：CoreDNS
- **指标服务器**：Metrics Server

## 安装指南

### 单节点安装

#### Linux/macOS
```bash
# 下载并安装 k3s
curl -sfL https://get.k3s.io | sh -

# 检查状态
sudo systemctl status k3s

# 获取 kubeconfig
sudo cat /etc/rancher/k3s/k3s.yaml
```

#### Windows (WSL2)
```bash
# 在 WSL2 中安装
curl -sfL https://get.k3s.io | sh -

# 或使用 Docker
docker run --privileged -d --name k3s-server -p 6443:6443 rancher/k3s:latest
```

### 多节点集群

#### 主节点 (Server)
```bash
# 安装主节点
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644

# 获取节点令牌
sudo cat /var/lib/rancher/k3s/server/node-token
```

#### 工作节点 (Agent)
```bash
# 加入集群
curl -sfL https://get.k3s.io | K3S_URL=https://myserver:6443 K3S_TOKEN=mynodetoken sh -
```

## 配置选项

### 常用启动参数

```bash
# 禁用 Traefik
curl -sfL https://get.k3s.io | sh -s - --disable traefik

# 使用外部数据库
curl -sfL https://get.k3s.io | sh -s - --datastore-endpoint="mysql://username:password@tcp(hostname:3306)/database"

# 自定义集群 CIDR
curl -sfL https://get.k3s.io | sh -s - --cluster-cidr=10.42.0.0/16 --service-cidr=10.43.0.0/16

# 启用 Docker 作为容器运行时
curl -sfL https://get.k3s.io | sh -s - --docker
```

### 配置文件示例

创建 `/etc/rancher/k3s/config.yaml`：

```yaml
# 服务器配置
write-kubeconfig-mode: "0644"
tls-san:
  - "my-kubernetes-domain.com"
  - "another-kubernetes-domain.com"
disable:
  - traefik
  - servicelb
cluster-init: true
```

## 基本使用

### kubectl 命令

```bash
# k3s 内置 kubectl
sudo k3s kubectl get nodes
sudo k3s kubectl get pods --all-namespaces

# 或者配置 kubeconfig
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl get nodes
```

### 部署应用示例

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
```

```bash
# 部署应用
kubectl apply -f nginx-deployment.yaml

# 查看服务
kubectl get services
```

## 网络配置

### Flannel 网络

K3s 默认使用 Flannel 作为 CNI 插件：

```bash
# 查看网络配置
kubectl get nodes -o wide
kubectl get pods -n kube-system | grep flannel
```

### 自定义 CNI

```bash
# 禁用 Flannel，使用自定义 CNI
curl -sfL https://get.k3s.io | sh -s - --flannel-backend=none --disable-network-policy

# 安装 Calico
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

## 存储配置

### Local Path Provisioner

```yaml
# 使用本地存储
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-path-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 2Gi
```

### 外部存储

```bash
# 安装 Longhorn 分布式存储
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/master/deploy/longhorn.yaml
```

## 监控和日志

### 内置监控

```bash
# 查看资源使用情况
kubectl top nodes
kubectl top pods --all-namespaces
```

### Prometheus 监控

```bash
# 安装 Prometheus
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack
```

### 日志管理

```bash
# 查看 k3s 日志
sudo journalctl -u k3s -f

# 查看 Pod 日志
kubectl logs -f deployment/nginx-deployment
```

## 高可用配置

### 嵌入式 etcd 集群

```bash
# 第一个节点
curl -sfL https://get.k3s.io | sh -s - --cluster-init

# 其他主节点
curl -sfL https://get.k3s.io | sh -s - --server https://first-server:6443 --token <token>
```

### 外部数据库

```bash
# 使用 MySQL
curl -sfL https://get.k3s.io | sh -s - --datastore-endpoint="mysql://user:pass@tcp(mysql-host:3306)/k3s"

# 使用 PostgreSQL
curl -sfL https://get.k3s.io | sh -s - --datastore-endpoint="postgres://user:pass@postgres-host:5432/k3s"
```

## 安全配置

### TLS 证书

```bash
# 自定义 TLS SAN
curl -sfL https://get.k3s.io | sh -s - --tls-san your-domain.com --tls-san your-ip
```

### RBAC 配置

```yaml
# rbac-example.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```
