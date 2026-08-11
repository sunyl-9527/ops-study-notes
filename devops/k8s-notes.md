# Kubernetes 入门学习笔记
> 适用对象：已掌握 Docker 与 Compose，希望进入容器编排阶段的学习者  
> 定位：对应 [cloud-ops-roadmap.md](./cloud-ops-roadmap.md) 第七阶段，覆盖核心对象、应用发布、配置存储与基础排障
## 目录
1. [Kubernetes 是什么](#1-kubernetes-是什么)
2. [集群架构](#2-集群架构)
3. [本地实验环境](#3-本地实验环境)
4. [Pod](#4-pod)
5. [Deployment](#5-deployment)
6. [Service](#6-service)
7. [Ingress](#7-ingress)
8. [ConfigMap 与 Secret](#8-configmap-与-secret)
9. [PV、PVC 与 StorageClass](#9-pvpvc-与-storageclass)
10. [StatefulSet 简介](#10-statefulset-简介)
11. [健康检查](#11-健康检查)
12. [命名空间、资源限额与 QoS](#12-命名空间资源限额与-qos)
13. [kubectl 常用命令速查](#13-kubectl-常用命令速查)
14. [排障入门](#14-排障入门)
15. [Helm 简介](#15-helm-简介)
16. [学习路线建议](#16-学习路线建议)
17. [动手实践](#17-动手实践)
18. [实践检查清单](#18-实践检查清单)
## 1. Kubernetes 是什么
Kubernetes，简称 K8s，是一个容器编排平台。它不负责制作镜像，而是把镜像运行在一组节点上，并持续让实际状态接近期望状态。
例如声明“运行 3 个 Nginx 副本”后，K8s 会选择节点创建 Pod；某个 Pod 消失时，控制器会自动补建。
### 1.1 主要解决的问题
- 在多台节点间调度容器
- 自动维持副本和故障恢复
- 为不断变化的 Pod 提供稳定入口
- 执行滚动更新与回滚
- 管理配置、密钥和持久化存储
- 通过健康检查控制重启和流量
- 以统一对象模型支持扩缩容、监控和自动化
### 1.2 与 Docker Compose 对比
| 能力 | Docker Compose | Kubernetes |
|---|---|---|
| 运行范围 | 通常是单台 Docker 主机 | 多节点集群 |
| 调度 | 当前主机直接运行 | 调度器选择节点 |
| 故障恢复 | 单机范围，能力有限 | 控制器持续协调状态 |
| 服务发现 | Compose 网络和服务名 | Service 与集群 DNS |
| 发布 | 基础 Compose 不擅长滚动更新 | Deployment 原生支持更新、回滚 |
| 配置与密钥 | 环境变量、文件、secrets | ConfigMap、Secret |
| 存储 | 主机卷或外部卷 | PV、PVC、StorageClass |
```text
Docker：制作镜像、运行容器
  ├─ Compose：在一台主机组织多个容器
  └─ Kubernetes：在集群声明、调度和维护应用
```
K8s 不会取代 Docker 基础。镜像是否存在、容器命令是否正确、端口是否监听、数据是否持久化，仍是排障重点。
## 2. 集群架构
集群由控制平面和一个或多个工作节点组成。
### 2.1 文字架构图
```text
                           kubectl / Helm
                                  │
                                  ▼
┌────────────────────────── 控制平面 ──────────────────────────┐
│ kube-apiserver：统一接口、认证鉴权、对象读写                  │
│   ├─ etcd：保存集群状态                                      │
│   ├─ kube-scheduler：为 Pod 选择节点                          │
│   └─ kube-controller-manager：运行控制器、协调期望状态        │
└──────────────────────────────┬────────────────────────────────┘
                               │
              ┌────────────────┴────────────────┐
              ▼                                 ▼
┌────────── 工作节点一 ──────────┐  ┌────────── 工作节点二 ──────────┐
│ kubelet      kube-proxy       │  │ kubelet      kube-proxy       │
│ 容器运行时   CNI 网络插件      │  │ 容器运行时   CNI 网络插件      │
│ Pod ─ 容器   Pod ─ 容器        │  │ Pod ─ 容器   Pod ─ 容器        │
└───────────────────────────────┘  └───────────────────────────────┘
```
### 2.2 控制平面组件
| 组件 | 作用 |
|---|---|
| `kube-apiserver` | 所有组件交互入口，处理认证、鉴权、准入和对象读写 |
| `etcd` | 保存集群对象与状态；生产环境必须保护和备份 |
| `kube-scheduler` | 根据资源、亲和性、污点等条件为 Pod 选择节点 |
| `kube-controller-manager` | 运行副本、节点等控制器，协调实际与期望状态 |
| `cloud-controller-manager` | 对接云厂商节点、负载均衡和存储能力 |
### 2.3 节点组件
| 组件 | 作用 |
|---|---|
| `kubelet` | 接收 Pod 目标状态并调用容器运行时 |
| 容器运行时 | 拉取镜像并运行容器，常见为 `containerd` |
| `kube-proxy` | 维护 Service 转发规则，部分网络方案会替代它 |
| CNI 插件 | 为 Pod 分配地址并实现网络，例如 Calico、Cilium |
K8s 的核心是声明式协调：提交 YAML 后，控制器不断观察差异并创建、删除或更新资源。因此应把 YAML 纳入版本控制。
## 3. 本地实验环境
本地可选择 Minikube 或 kind。两者适合学习，不代表生产集群部署方式。
### 3.1 前置条件与 kubectl
建议准备 2 核处理器、4 GiB 可用内存、Docker 或受支持的虚拟化环境，并确认能访问镜像仓库。
Linux 安装示例，`<稳定版本>` 替换为当前稳定版本，如 `v1.x.y`：
```bash
curl -LO "https://dl.k8s.io/release/<稳定版本>/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```
Windows 与 macOS：
```powershell
winget install --id Kubernetes.kubectl
kubectl version --client
```
```bash
brew install kubectl
kubectl version --client
```
### 3.2 自动补全
```bash
# Bash
source <(kubectl completion bash)
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
# Zsh
source <(kubectl completion zsh)
echo 'source <(kubectl completion zsh)' >> ~/.zshrc
```
```powershell
# PowerShell；需要永久生效时写入当前用户的 $PROFILE
kubectl completion powershell | Out-String | Invoke-Expression
```
### 3.3 Minikube
Minikube 通常创建本地单节点集群，内置插件适合练习 Ingress。
```bash
minikube start --driver=docker --cpus=2 --memory=4096
minikube status
kubectl get nodes -o wide
minikube addons enable ingress
minikube stop
minikube delete
```
### 3.4 kind
kind 把 Kubernetes 节点作为 Docker 容器运行，创建快，适合多节点实验和自动化测试。
```bash
go install sigs.k8s.io/kind@latest
kind create cluster --name k8s-lab
kubectl cluster-info --context kind-k8s-lab
kind delete cluster --name k8s-lab
```
多节点配置 `kind-config.yaml`：
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```
```bash
kind create cluster --name multi-lab --config kind-config.yaml
kubectl get nodes
```
| 场景 | 建议 |
|---|---|
| 快速启用内置 Ingress、仪表盘 | Minikube |
| 频繁创建销毁、多节点、自动化测试 | kind |
操作前总是确认目标集群：
```bash
kubectl config get-contexts
kubectl config current-context
kubectl config use-context <上下文名称>
kubectl cluster-info
```
## 4. Pod
Pod 是 K8s 最小调度单元，不是容器本身。一个 Pod 可包含一个或多个紧密协作的容器。
同一 Pod 内的容器会被调度到同一节点，共享 Pod 地址和端口空间，可通过 `localhost` 通信，也可挂载同一卷。常见模式仍是一个 Pod 一个业务容器。
```text
Pod
├─ 共享网络：一个 Pod 地址
├─ 共享卷：按声明挂载
├─ 主业务容器
└─ 可选辅助容器
```
Pod 重建后地址可能变化，不能把 Pod 地址当作稳定入口，应使用 Service。
### 4.1 Pod YAML
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: default
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.27-alpine
      ports:
        - name: http
          containerPort: 80
      resources:
        requests:
          cpu: 50m
          memory: 64Mi
        limits:
          cpu: 200m
          memory: 128Mi
  restartPolicy: Always
```
```bash
kubectl apply -f nginx-pod.yaml
kubectl get pod nginx-pod -o wide
kubectl describe pod nginx-pod
```
### 4.2 逐段理解
- `apiVersion`：对象接口版本；Pod 属于核心组，使用 `v1`
- `kind`：对象类型
- `metadata`：名称、命名空间、标签、注解等元数据
- `spec`：期望状态，包含镜像、端口、资源和重启策略
- `status`：系统维护的实际状态，不手工写入声明文件
- `labels`：供筛选和对象关联使用
- `ports`：说明容器端口，不等于自动对外暴露
```bash
kubectl get pod nginx-pod -o yaml
kubectl get pod nginx-pod -o jsonpath='{.status.phase}'
```
直接创建的 Pod 被删除后没有控制器补建。长期无状态应用通常使用 Deployment。
## 5. Deployment
Deployment 管理 ReplicaSet，ReplicaSet 维持 Pod 副本，适合 Nginx、前端和接口等无状态服务。
```text
Deployment → ReplicaSet → Pod、Pod、Pod
```
### 5.1 完整 YAML
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: default
spec:
  replicas: 3
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
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
          image: nginx:1.27-alpine
          ports:
            - name: http
              containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 3
            periodSeconds: 5
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
```
`selector.matchLabels` 必须与 Pod 模板标签匹配。
### 5.2 副本、更新与回滚
```bash
kubectl apply -f nginx-deployment.yaml
kubectl get deployment,replicaset,pod
kubectl scale deployment nginx --replicas=5
kubectl set image deployment/nginx nginx=nginx:1.28-alpine
kubectl rollout status deployment/nginx
kubectl rollout history deployment/nginx
kubectl rollout history deployment/nginx --revision=2
kubectl rollout undo deployment/nginx
kubectl rollout undo deployment/nginx --to-revision=2
kubectl rollout status deployment/nginx
```
`maxUnavailable: 1` 表示更新时最多少 1 个可用副本；`maxSurge: 1` 表示最多临时多建 1 个 Pod。命令式修改后，应把最终期望状态同步回 YAML。
## 6. Service
Service 根据标签选择 Pod，并提供稳定的虚拟地址和 DNS 名称。
| 类型 | 范围 | 用途 |
|---|---|---|
| `ClusterIP` | 集群内部，默认类型 | 服务间调用 |
| `NodePort` | 每个节点的高位端口 | 本地实验、临时暴露 |
| `LoadBalancer` | 外部负载均衡地址 | 云环境对外服务 |
### 6.1 ClusterIP YAML
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - name: http
      port: 80
      targetPort: http
      protocol: TCP
```
```bash
kubectl apply -f nginx-service.yaml
kubectl get service nginx
kubectl get endpointslices -l kubernetes.io/service-name=nginx
kubectl run curl-test --rm -it --restart=Never --image=curlimages/curl -- curl http://nginx
```
`port` 是 Service 端口，`targetPort` 是 Pod 端口或命名端口。
### 6.2 其他类型 YAML 关键差异
NodePort：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: http
      nodePort: 30080
```
LoadBalancer：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-public
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: http
```
多数集群默认 NodePort 范围为 `30000-32767`。本地集群没有负载均衡实现时，LoadBalancer 外部地址可能一直等待。
Service 无后端时检查标签与就绪状态：
```bash
kubectl get pods -l app=nginx --show-labels
kubectl get endpointslices -l kubernetes.io/service-name=nginx
```
## 7. Ingress
Ingress 声明 HTTP/HTTPS 路由，可按域名和路径把请求转发到 Service。只有集群安装了 Ingress Controller，规则才会生效。
```text
客户端 → Ingress Controller → Service → Pod
```
Minikube 可启用 NGINX Ingress Controller：
```bash
minikube addons enable ingress
kubectl get pods -n ingress-nginx
```
也可通过官方 Helm 仓库安装：
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```
### 7.1 Ingress YAML
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
  namespace: default
spec:
  ingressClassName: nginx
  rules:
    - host: lab.example.test
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx
                port:
                  number: 80
```
```bash
kubectl apply -f nginx-ingress.yaml
kubectl get ingress
kubectl describe ingress nginx
curl --resolve lab.example.test:80:<入口地址> http://lab.example.test/
```
生产环境还要考虑 DNS、TLS 证书、入口高可用、真实客户端地址和日志。
## 8. ConfigMap 与 Secret
ConfigMap 保存非敏感配置，Secret 保存口令、令牌和证书等敏感数据，使配置与镜像分离。
### 8.1 创建 ConfigMap
```bash
kubectl create configmap app-config \
  --from-literal=APP_MODE=production \
  --from-literal=LOG_LEVEL=info
kubectl create configmap nginx-page --from-file=index.html=<本地文件路径>
```
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_MODE: production
  LOG_LEVEL: info
  app.properties: |
    server.port=8080
    feature.enabled=true
```
### 8.2 挂载为环境变量和文件
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "env | sort; cat /etc/app/app.properties; sleep 3600"]
      envFrom:
        - configMapRef:
            name: app-config
      volumeMounts:
        - name: config-files
          mountPath: /etc/app
          readOnly: true
  volumes:
    - name: config-files
      configMap:
        name: app-config
```
环境变量通常需重建 Pod 才生效；卷内配置可延迟更新，但应用是否重载取决于应用本身。
### 8.3 Secret
```bash
kubectl create secret generic app-secret \
  --from-literal=DB_USER=<数据库用户> \
  --from-literal=DB_PASSWORD=<数据库密码>
```
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:
  DB_USER: <数据库用户>
  DB_PASSWORD: <数据库密码>
```
环境变量与文件挂载示例：
```yaml
env:
  - name: DB_USER
    valueFrom:
      secretKeyRef:
        name: app-secret
        key: DB_USER
volumeMounts:
  - name: secret-files
    mountPath: /run/secrets
    readOnly: true
volumes:
  - name: secret-files
    secret:
      secretName: app-secret
```
Secret 默认只是 Base64 编码，不是加密。生产环境应限制 RBAC、启用静态数据加密、避免明文进入 Git，并考虑外部密钥管理系统。
## 9. PV、PVC 与 StorageClass
容器文件系统可能随容器重建丢失。持久数据需要存储抽象：
```text
Pod
 └─ 挂载 PVC：应用提出容量与访问模式需求
       └─ 绑定 PV：集群中的实际存储资源
             └─ 本地盘 / 网络存储 / 云硬盘
StorageClass ─ 定义如何动态创建 PV、参数和回收策略
```
| 对象 | 作用 |
|---|---|
| PV | 集群提供的一块持久化存储 |
| PVC | 应用对存储的申请 |
| StorageClass | 动态供应存储的类型与参数 |
### 9.1 PVC 与使用示例
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: <存储类名称>
```
若使用默认 StorageClass，可删除 `storageClassName`。Pod 引用：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: storage-demo
spec:
  containers:
    - name: writer
      image: busybox:1.36
      command: ["sh", "-c", "date >> /data/history.log; sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: app-data
```
```bash
kubectl get storageclass
kubectl get pvc,pv
kubectl describe pvc app-data
```
常见访问模式有 `ReadWriteOnce`、`ReadOnlyMany`、`ReadWriteMany`。删除 PVC 前要确认备份和 PV 的 `Delete` 或 `Retain` 回收策略。
## 10. StatefulSet 简介
StatefulSet 管理需要稳定身份、独立稳定存储或有序启停的应用。
| 对比项 | Deployment | StatefulSet |
|---|---|---|
| Pod 名称 | 随机后缀、可互换 | 稳定序号，如 `db-0` |
| 网络身份 | 不强调稳定 | 常配合无头 Service 提供稳定域名 |
| 存储 | 不强调每副本独立卷 | 可为每个 Pod 创建独立 PVC |
| 启停 | 通常并行 | 默认按顺序 |
| 场景 | Web、接口 | 数据库、需要成员身份的集群 |
满足以下需求时再考虑 StatefulSet：固定名称、每副本独立卷、按序启动、固定成员身份。它不会替应用完成数据库主从配置；扩缩容和升级仍须遵循应用自身流程并先备份。
## 11. 健康检查
| 探针 | 判断内容 | 失败行为 |
|---|---|---|
| `livenessProbe` | 容器是否失去响应 | 重启容器 |
| `readinessProbe` | 是否可接收流量 | 从 Service 后端暂时移除 |
| `startupProbe` | 慢启动应用是否已启动 | 完成前暂停其他探针，持续失败则重启 |
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-with-probes
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-with-probes
  template:
    metadata:
      labels:
        app: web-with-probes
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - name: http
              containerPort: 80
          startupProbe:
            httpGet:
              path: /
              port: http
            periodSeconds: 2
            failureThreshold: 30
          readinessProbe:
            httpGet:
              path: /
              port: http
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: http
            periodSeconds: 10
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
```
探针还支持 TCP、命令和 gRPC。探针应快速稳定；不要用存活探针检测临时外部依赖，避免依赖抖动触发集体重启。
## 12. 命名空间、资源限额与 QoS
Namespace 用于按环境、团队或项目分组资源，但本身不是完整安全边界。
```bash
kubectl create namespace k8s-lab
kubectl get namespaces
kubectl config set-context --current --namespace=k8s-lab
```
Node、PV、StorageClass 等属于集群级对象，不在命名空间内。
### 12.1 requests 与 limits
- `requests`：调度器用于评估和预留的请求量
- `limits`：容器可用上限；CPU 超限通常被限速，内存超限可能被终止
- `100m` CPU 表示 0.1 核；内存应使用 `Mi`、`Gi`
```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```
LimitRange 可设置默认值，ResourceQuota 可限制命名空间总量：
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: lab-quota
  namespace: k8s-lab
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 2Gi
    limits.cpu: "4"
    limits.memory: 4Gi
    pods: "20"
```
### 12.2 QoS
| 等级 | 条件概括 | 内存紧张时相对风险 |
|---|---|---|
| `Guaranteed` | 每个容器都设置 CPU、内存请求与限制，且请求等于限制 | 较低 |
| `Burstable` | 至少设置一项，但不满足 Guaranteed | 中等 |
| `BestEffort` | 未设置 CPU 和内存请求、限制 | 较高 |
```bash
kubectl get pod <Pod名称> -o jsonpath='{.status.qosClass}'
```
## 13. kubectl 常用命令速查
### 13.1 查看
| 目标 | 命令 |
|---|---|
| 节点 | `kubectl get nodes -o wide` |
| 全部命名空间 Pod | `kubectl get pods -A -o wide` |
| 按标签筛选 | `kubectl get pods -l app=nginx` |
| 持续观察 | `kubectl get pods -w` |
| 对象详情和事件 | `kubectl describe pod <名称>` |
| 完整 YAML | `kubectl get deployment <名称> -o yaml` |
| 工作负载 | `kubectl get deploy,sts,ds -A` |
| 服务入口 | `kubectl get svc,ingress -A` |
| 字段说明 | `kubectl explain deployment.spec.strategy` |
### 13.2 创建与修改
| 目标 | 命令 |
|---|---|
| 应用或删除 YAML | `kubectl apply -f <文件>` / `kubectl delete -f <文件>` |
| 生成 YAML | `kubectl create deployment demo --image=nginx --dry-run=client -o yaml` |
| 查看差异 | `kubectl diff -f <文件>` |
| 扩缩容 | `kubectl scale deployment <名称> --replicas=3` |
| 更新镜像 | `kubectl set image deployment/<名称> <容器>=<镜像>` |
| 编辑对象 | `kubectl edit deployment <名称>` |
### 13.3 调试
| 目标 | 命令 |
|---|---|
| 执行命令 | `kubectl exec <Pod> -- <命令>` |
| 进入容器 | `kubectl exec -it <Pod> -c <容器> -- sh` |
| 临时调试 Pod | `kubectl run debug --rm -it --restart=Never --image=busybox:1.36 -- sh` |
| 端口转发 | `kubectl port-forward service/<服务> 8080:80` |
| 按时间查看事件 | `kubectl get events --sort-by=.metadata.creationTimestamp` |
| 资源使用 | `kubectl top pods` |
### 13.4 日志
| 目标 | 命令 |
|---|---|
| 当前日志 | `kubectl logs <Pod>` |
| 持续日志 | `kubectl logs -f <Pod>` |
| 指定容器 | `kubectl logs <Pod> -c <容器>` |
| 上一次容器日志 | `kubectl logs <Pod> --previous` |
| 最近 100 行 | `kubectl logs <Pod> --tail=100` |
| 标签下所有容器 | `kubectl logs -l app=nginx --all-containers=true` |
`-n` 指定命名空间，`-A` 表示全部，`-o wide` 显示更多字段，`-l` 按标签筛选。
## 14. 排障入门
不要先删除 Pod 碰运气。按以下流程缩小范围：
```text
确认上下文和命名空间
  → get pod -o wide 看状态、节点、重启次数
  → describe pod 看 Events、探针、挂载
  → logs 看当前日志
  → logs --previous 看上次崩溃日志
  → 检查 Deployment、ConfigMap、Secret、PVC、Service
  → 修正 YAML，apply 后观察恢复
```
```bash
kubectl config current-context
kubectl get pod <Pod> -n <命名空间> -o wide
kubectl describe pod <Pod> -n <命名空间>
kubectl logs <Pod> -n <命名空间> -c <容器>
kubectl logs <Pod> -n <命名空间> -c <容器> --previous
kubectl get events -n <命名空间> --sort-by=.metadata.creationTimestamp
```
### 14.1 常见状态
| 状态 | 常见原因 | 排查命令 |
|---|---|---|
| `Pending` | 资源不足、PVC 未绑定、节点条件不符、污点未容忍 | `describe pod`、`get nodes`、`get pvc` |
| `ImagePullBackOff` | 镜像或标签错误、仓库不可达、凭据错误 | `describe pod`、`get secret` |
| `CrashLoopBackOff` | 主进程退出、配置错误、权限问题、探针失败 | `logs`、`logs --previous`、`describe pod` |
| `CreateContainerConfigError` | ConfigMap、Secret 或键不存在 | `describe pod`、`get configmap,secret` |
| `Running` 未就绪 | 就绪探针失败、端口错误 | `describe pod`、`logs` |
| `OOMKilled` | 内存限制过低或节点压力 | `describe pod`、`top pod` |
### 14.2 分状态排查
Pending：
```bash
kubectl describe pod <Pod>
kubectl get nodes
kubectl describe node <节点>
kubectl get pvc
```
重点看 `Insufficient cpu`、`Insufficient memory`、PVC 未绑定、节点选择器和污点事件。
ImagePullBackOff：
```bash
kubectl describe pod <Pod>
kubectl get pod <Pod> -o jsonpath='{.spec.containers[*].image}'
kubectl get secret -n <命名空间>
```
检查镜像名称、标签、网络、DNS、私有仓库凭据和 `imagePullSecrets`。
CrashLoopBackOff：
```bash
kubectl logs <Pod> --all-containers=true
kubectl logs <Pod> --all-containers=true --previous
kubectl describe pod <Pod>
kubectl get pod <Pod> -o jsonpath='{.status.containerStatuses[*].lastState}'
```
常见原因是进程立即退出、参数或配置错误、缺少依赖、权限失败、探针过严、内存不足。
Service 不通时按链路检查：
```bash
kubectl get service <服务> -o yaml
kubectl get endpointslices -l kubernetes.io/service-name=<服务>
kubectl get pods -l <选择器> --show-labels
```
没有后端地址时先检查标签和 readinessProbe；有后端再查容器监听端口和网络策略。
## 15. Helm 简介
Helm 是 Kubernetes 应用包管理工具。Chart 是带模板的资源包，安装后的实例称为 Release。
```text
Chart/
├─ Chart.yaml       元数据
├─ values.yaml      默认配置
├─ templates/       资源模板
└─ charts/          依赖
Chart + values → 渲染 YAML → 安装为 Release
```
```bash
helm repo add <仓库名> <仓库地址>
helm repo update
helm search repo <关键词>
helm show values <仓库名>/<Chart>
helm install <发布名> <仓库名>/<Chart> \
  -n <命名空间> --create-namespace -f <值文件>
helm upgrade --install <发布名> <仓库名>/<Chart> \
  -n <命名空间> -f <值文件>
helm list -A
helm history <发布名> -n <命名空间>
helm rollback <发布名> <修订号> -n <命名空间>
helm uninstall <发布名> -n <命名空间>
```
安装前使用 `helm lint`、`helm template` 或 `--dry-run --debug` 检查将创建的对象、镜像和权限。
## 16. 学习路线建议
1. 查看节点、命名空间和接口资源
2. 创建 Pod，理解 Pod 与容器
3. 用 Deployment 管理无状态副本
4. 用 Service 提供稳定入口
5. 用 ConfigMap、Secret 分离配置
6. 用 Ingress 练习域名和路径路由
7. 加入探针和资源限制
8. 练习滚动更新、失败更新和回滚
9. 故意制造镜像、命令、探针和存储故障
10. 使用 PVC，再了解 StatefulSet
11. 使用 Helm 安装实验应用
12. 继续学习 RBAC、网络策略、扩缩容、监控和高可用
每次操作都回答：期望状态在哪里、哪个控制器负责、对象如何关联、出错应看状态/事件/日志还是依赖。
## 17. 动手实践
> 使用本地实验集群，所有资源放入 `k8s-lab` 命名空间。每步执行后对照预期结果。
### 17.1 检查集群并创建命名空间
```bash
kubectl config current-context
kubectl cluster-info
kubectl get nodes
kubectl create namespace k8s-lab
```
**预期结果**：至少一个节点为 `Ready`，命名空间创建成功。
### 17.2 部署 Nginx
保存为 `01-nginx.yaml`：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-lab
  namespace: k8s-lab
spec:
  replicas: 3
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: nginx-lab
  template:
    metadata:
      labels:
        app: nginx-lab
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - name: http
              containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 2
            periodSeconds: 5
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-lab
  namespace: k8s-lab
spec:
  type: ClusterIP
  selector:
    app: nginx-lab
  ports:
    - name: http
      port: 80
      targetPort: http
```
```bash
kubectl apply -f 01-nginx.yaml
kubectl rollout status deployment/nginx-lab -n k8s-lab
kubectl get deployment,pod,service -n k8s-lab -o wide
kubectl get endpointslices -n k8s-lab \
  -l kubernetes.io/service-name=nginx-lab
```
**预期结果**：Deployment 为 `3/3`，3 个 Pod 为 `Running`，Service 有 3 个后端地址。
集群内验证：
```bash
kubectl run curl-test -n k8s-lab --rm -it --restart=Never \
  --image=curlimages/curl -- curl -s http://nginx-lab
```
**预期结果**：输出 Nginx 默认欢迎页 HTML。
### 17.3 用 ConfigMap 注入页面
保存为 `02-page.yaml`：
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-page
  namespace: k8s-lab
data:
  index.html: |
    <!doctype html>
    <html lang="zh-CN">
      <head><meta charset="utf-8"><title>Kubernetes 实验</title></head>
      <body>
        <h1>你好，Kubernetes！</h1>
        <p>此页面来自 ConfigMap。</p>
      </body>
    </html>
```
```bash
kubectl apply -f 02-page.yaml
```
在 `01-nginx.yaml` 的容器中加入：
```yaml
          volumeMounts:
            - name: web-content
              mountPath: /usr/share/nginx/html
              readOnly: true
```
在 Pod 模板 `spec` 中与 `containers` 同级加入：
```yaml
      volumes:
        - name: web-content
          configMap:
            name: nginx-page
```
```bash
kubectl apply -f 01-nginx.yaml
kubectl rollout status deployment/nginx-lab -n k8s-lab
kubectl port-forward service/nginx-lab 8080:80 -n k8s-lab
```
另一终端执行：
```bash
curl http://127.0.0.1:8080
```
**预期结果**：页面包含“你好，Kubernetes！”和“此页面来自 ConfigMap”。完成后按 `Ctrl+C` 停止转发。
### 17.4 滚动更新与回滚
```bash
kubectl set image deployment/nginx-lab \
  nginx=nginx:1.28-alpine -n k8s-lab
kubectl annotate deployment nginx-lab \
  kubernetes.io/change-cause="升级 Nginx 镜像" \
  -n k8s-lab --overwrite
kubectl rollout status deployment/nginx-lab -n k8s-lab
kubectl rollout history deployment/nginx-lab -n k8s-lab
```
**预期结果**：旧 Pod 被逐步替换，最终仍有 3 个就绪副本。若该标签不存在，替换为仓库中确认存在的版本。
```bash
kubectl rollout undo deployment/nginx-lab -n k8s-lab
kubectl rollout status deployment/nginx-lab -n k8s-lab
kubectl get deployment nginx-lab -n k8s-lab \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```
**预期结果**：镜像恢复到上一版本，所有副本重新就绪。把最终期望镜像同步回 YAML。
### 17.5 制造并排查 CrashLoopBackOff
保存为 `03-crash.yaml`：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crash-demo
  namespace: k8s-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: crash-demo
  template:
    metadata:
      labels:
        app: crash-demo
    spec:
      containers:
        - name: crash-demo
          image: busybox:1.36
          command: ["sh", "-c", "echo '程序即将异常退出'; exit 1"]
          resources:
            requests:
              cpu: 10m
              memory: 16Mi
            limits:
              cpu: 50m
              memory: 32Mi
```
```bash
kubectl apply -f 03-crash.yaml
kubectl get pods -n k8s-lab -l app=crash-demo -w
```
**预期结果**：重启次数增加，状态逐渐进入 `CrashLoopBackOff`。按 `Ctrl+C` 停止观察。
```bash
kubectl describe pod -n k8s-lab -l app=crash-demo
kubectl logs -n k8s-lab -l app=crash-demo
kubectl logs -n k8s-lab -l app=crash-demo --previous
```
**预期结果**：日志出现“程序即将异常退出”，详情显示非零退出码和反复重启。
修改 `03-crash.yaml` 的命令后重新应用：
```yaml
          command: ["sh", "-c", "echo '程序已修复'; sleep 3600"]
```
```bash
kubectl apply -f 03-crash.yaml
kubectl rollout status deployment/crash-demo -n k8s-lab
kubectl get pods -n k8s-lab -l app=crash-demo
```
**预期结果**：新 Pod 为 `Running`，不再持续重启。
### 17.6 清理
```bash
kubectl get all,configmap -n k8s-lab
kubectl delete namespace k8s-lab
kubectl get namespace k8s-lab
```
**预期结果**：实验资源随命名空间删除。若该本地集群不再使用，再执行对应工具的 `minikube delete` 或 `kind delete cluster --name <集群名>`。
## 18. 实践检查清单
- [ ] 能说明 K8s 相比 Compose 解决的编排问题
- [ ] 能说出控制平面和节点主要组件
- [ ] 能用 Minikube 或 kind 创建本地集群
- [ ] 能确认 kubectl 上下文和命名空间
- [ ] 能解释 Pod 与容器的关系并阅读基础 YAML
- [ ] 能用 Deployment 管理无状态副本
- [ ] 能区分 ClusterIP、NodePort、LoadBalancer
- [ ] 能说明 Ingress 与 Ingress Controller 的区别
- [ ] 能用 ConfigMap、Secret 注入环境变量或文件
- [ ] 知道 Secret 的 Base64 不是加密
- [ ] 能解释 PV、PVC、StorageClass 的关系
- [ ] 能判断何时使用 StatefulSet
- [ ] 能区分三种健康探针
- [ ] 能设置 requests、limits 并理解 QoS
- [ ] 能使用 `get`、`describe`、`logs`、`exec`、`port-forward`
- [ ] 能排查 Pending、ImagePullBackOff、CrashLoopBackOff
- [ ] 能完成滚动更新、查看历史并回滚
- [ ] 能解释 Helm Chart、values 与 Release
- [ ] 能部署 Nginx、通过 Service 访问并注入 ConfigMap
- [ ] 能故意制造故障、记录证据、修复并清理环境
