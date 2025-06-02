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
  name: simple-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: localhost
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: simple-webserver-svc
            port:
              number: 80 
```

Nginx Ingress Contoller 配置：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: nginx-ingress-controller
rules:
  - apiGroups: [""]
    resources: ["configmaps", "endpoints", "nodes", "pods", "secrets"]
    verbs: ["list", "watch"]
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["extensions", "networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "patch"]
  - apiGroups: ["extensions", "networking.k8s.io"]
    resources: ["ingresses/status"]
    verbs: ["update"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingressclasses"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "create", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nginx-ingress-controller
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-controller
    namespace: ingress-nginx
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: nginx-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nginx-ingress-controller
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-controller
    namespace: ingress-nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
spec:
  selector:
    matchLabels:
      app: nginx-ingress-controller
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx-ingress-controller
    spec:
      serviceAccountName: nginx-ingress-controller
      containers:
        - name: nginx-ingress-controller
          image: k8s.gcr.io/ingress-nginx/controller:v1.1.1
          args:
            - /nginx-ingress-controller
            - --publish-service=ingress-nginx/ingress-nginx-controller
            - --election-id=ingress-controller-leader
            - --ingress-class=nginx
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
---
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  type: LoadBalancer
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
    - name: https
      port: 443
      targetPort: 443
      protocol: TCP
  selector:
    app: nginx-ingress-controller 
```

随便起一个后端服务：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webserver
  labels:
    app: simple-webserver
spec:
  containers:
  - name: python-server
    image: python:2.7
    command: ["python", "-m", "SimpleHTTPServer", "8080"]
    ports:
    - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: simple-webserver-svc
spec:
  selector:
    app: simple-webserver
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
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

