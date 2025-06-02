# Ingress 和 NIC(Nginx Ingress Controller) 初探

## Ingress

Ingress 是 Kubernetes 中管理外部访问集群内部服务的 API 对象，主要提供 HTTP/HTTPS 路由、负载均衡、SSL/TLS 终止和基于名称的虚拟主机等功能。

### 配置定义

Ingress 的配置文件定义了如何将外部 HTTP/HTTPS 流量路由到集群内的 Service（后端 Pod）。

### 依赖控制器

Ingress 资源本身不生效，必须由 Ingress Controller 监听并执行规则。用户创建 Ingress 后，Controller 会将其转化为负载均衡器的实际配置

## Nginx Ingress Controller

Nginx Ingress Controller 是 Ingress 规则的具体实现者，它使用 Nginx 作为反向代理和负载均衡引擎。

### 核心职责

- 监控 Kubernetes API 中的 Ingress 资源变化。

- 动态生成 Nginx 配置文件（nginx.conf）。

- 执行 nginx -s reload。

- 提供负载均衡、SSL 终止等能力。

### 工作流程

1. 通过 Informer 监听 Ingress 资源变更。

2. 将变更事件加入工作队列（Workqueue）。

3. Controller 从队列获取事件，生成配置并触发 Nginx 重载。

## 最小可工作示例

Ingress 配置：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "15"
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /v1(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: v1-service
            port: 80
```

查看生成的Nginx配置：

```bash
kubectl exec -it <ingress-pod> -- cat /etc/nginx/nginx.conf
```

检查Annotation生效状态：

```bash
kubectl describe ingress <ingress-name> | grep Annotations
```

监控配置重载：

```bash
kubectl logs -f <ingress-pod> | grep "backend reload required"
```

