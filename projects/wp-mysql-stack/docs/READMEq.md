## 🧩 环境排障记录
# Kubernetes 部署踩坑记录：WordPress + MySQL

> 本文档记录了在 Kubernetes 集群上部署 WordPress + MySQL 应用过程中遇到的所有问题及解决方案，供后续参考。


### 1️⃣ 问题一：kubeadm init 超时 — pause 镜像拉取失败

**现象**
- `kubeadm init` 卡在 `Waiting for a healthy API server`，4 分钟后超时退出
- 报错：`context deadline exceeded`、`dial tcp ... connection refused`
- `sudo crictl ps -a` 无任何控制平面容器

**根因**
1. `registry.k8s.io/pause:3.10.1` 镜像无法拉取
2. containerd 的 `sandbox_image` 配置虽改为阿里云地址，但未生效
3. kubelet 的 `--pod-infra-container-image` 参数优先级更高，仍指向官方源

**解决方案**
```bash
# 手动拉取阿里云 pause 镜像并打上官方标签
sudo crictl pull registry.aliyuncs.com/google_containers/pause:3.10
sudo ctr -n k8s.io image tag registry.aliyuncs.com/google_containers/pause:3.10 registry.k8s.io/pause:3.10.1

# 重置并重新初始化
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes /var/lib/etcd
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --image-repository=registry.aliyuncs.com/google_containers
```

**经验教训**
- 手动预拉取 + 打标签是解决容器镜像拉取问题最可靠的方式
- `kubeadm init` 超时时，优先检查 pause 镜像是否存在：`crictl images | grep pause`
- 修改 containerd 配置后需用 `containerd config dump` 验证是否生效


### 2️⃣ 问题二：Flannel 网络插件 Pod 反复 CrashLoopBackOff

**现象**
- 节点 `NotReady`，Flannel Pod 反复重启
- 状态：`ImagePullBackOff` → `ErrImagePull` → `RunContainerError` → `CrashLoopBackOff`
- 最终报错：`exec: "/opt/bin/install-conf": no such file or directory`

**根因**
1. `ghcr.io` 被墙，Flannel 默认镜像无法拉取
2. 镜像标签与 DaemonSet 期望不一致
3. Flannel 镜像中不存在 `/opt/bin/install-conf` 脚本

**解决方案**
```bash
# 1. 从 DaoCloud 镜像源拉取可用版本并打标签
sudo crictl pull m.daocloud.io/docker.io/flannel/flannel:v0.25.7
sudo ctr -n k8s.io image tag m.daocloud.io/docker.io/flannel/flannel:v0.25.7 docker.io/flannel/flannel:v0.25.7
sudo ctr -n k8s.io image tag m.daocloud.io/docker.io/flannel/flannel:v0.25.7 ghcr.io/flannel-io/flannel:v0.28.9

# 2. 导入本地 v0.28.5 tar 包并打标签
sudo ctr -n k8s.io image import /home/k/flannel-v0.28.5.tar
sudo ctr -n k8s.io image tag ghcr.io/flannel-io/flannel:v0.28.5 docker.io/flannel/flannel:v0.28.5
sudo ctr -n k8s.io image tag ghcr.io/flannel-io/flannel:v0.28.5 ghcr.io/flannel-io/flannel:v0.28.9

# 3. 修改 DaemonSet，用 cp 命令替代不存在的 install-conf 脚本
kubectl patch daemonset kube-flannel-ds -n kube-flannel --type='json' -p='[
  {"op": "replace", "path": "/spec/template/spec/initContainers/1/command", "value": ["/bin/sh", "-c", "cp /etc/kube-flannel/cni-conf.json /etc/cni/net.d/10-flannel.conflist"]}
]'

# 4. 删除 Pod，让 DaemonSet 重建
kubectl delete pod -n kube-flannel --all --force --grace-period=0
```

**最终运行的版本**
- DaemonSet 实际使用的镜像：`docker.io/flannel/flannel:v0.28.5`（来自本地导入的 `flannel-v0.28.5.tar`）
- `ghcr.io/flannel-io/flannel:v0.28.9` 标签指向的是 `v0.28.5` 的镜像内容
- `v0.25.7` 仅作为备选保留在本地

**经验教训**
- 优先使用国内镜像源（如 `m.daocloud.io`）拉取被墙的镜像
- 镜像标签必须与 DaemonSet 期望一致，否则会报 `ErrImagePull`
- Init 容器报 `no such file or directory` 时，可直接用 `cp` 命令替代
- 修改 DaemonSet（`kubectl patch`）比反复 `kubeadm reset` 更高效
- 导出/导入镜像 tar 包是离线环境部署的可靠方法，可用 `ctr image import` 直接导入


### 3️⃣ 问题三：MySQL Pod ErrImagePull / ImagePullBackOff

**现象**
- `wp-db-0` 一直 Pending，状态反复 `ErrImagePull`、`ImagePullBackOff`
- `kubectl describe pod` 显示 Pod 调度到了 `k8s-node1`

**根因**
1. MySQL 镜像在 Master 节点上已拉取，但 Pod 被调度到 Worker 节点
2. Worker 节点没有 MySQL 镜像
3. `imagePullPolicy: Never` 强制只查本地，找不到镜像直接报错

**解决方案**
```bash
# 在 Worker 节点（k8s-node1）上导入镜像
sudo ctr -n k8s.io image import /home/k/mysql-5.7.tar
sudo ctr -n k8s.io image tag docker.io/library/mysql:5.7 docker.io/library/mysql:5.7
```

**经验教训**
- `kubectl get pods -o wide` 查看 Pod 所在节点，在对应节点准备镜像
- 节点之间共享 etcd 状态，但不共享容器镜像
- `imagePullPolicy: Never` 需配合本地镜像存在使用
- 使用 `imagePullPolicy: IfNotPresent` 可避免此类问题：本地有则用本地，无则从仓库拉取，更灵活
- 多节点集群中，建议搭建私有镜像仓库（如 Harbor 或 Docker Registry），统一分发镜像，避免每台节点手动导入


### 4️⃣ 问题四：WordPress 连接 MySQL 失败

**现象**
- 浏览器访问 WordPress 安装页面时报 `Error establishing a database connection`
- WordPress Pod 日志显示连接 MySQL 超时或拒绝

**根因**
1. MySQL 容器已启动，但 `wordpress` 数据库不存在
2. WordPress 的 `wp-config.php` 中配置了 `WORDPRESS_DB_NAME=wordpress`，但 MySQL 中未自动创建该数据库

**解决方案**
```bash
# 进入 MySQL 容器，手动创建 wordpress 数据库
kubectl exec -n wp-mysql-stack wp-db-0 -- mysql -uroot -p123456 -e "CREATE DATABASE IF NOT EXISTS wordpress;"
```

**经验教训**
- MySQL 初次启动时不会自动创建业务数据库，需要手动创建或通过初始化脚本完成
- WordPress 需要数据库存在才能完成安装，否则会报连接错误
- 使用 StatefulSet 部署有状态应用时，可配合 Init Container 或启动脚本完成数据库初始化
- 若使用 MySQL 官方镜像，可将初始化 SQL 脚本挂载到 `/docker-entrypoint-initdb.d/` 目录，实现首次启动自动执行


### 5️⃣ 问题五：Windows 无法访问 WordPress（NodePort 不通）

> **本问题的根因经过实际操作验证，修正了原始记录中“NodePort 只在 Worker 节点监听”的错误推断。**

**现象**
- 浏览器访问 `http://192.168.2.10:32074` 超时或拒绝连接
- `ping 192.168.2.10` 能通，但 `curl` 超时
- `sudo netstat -tlnp | grep 32074` 无输出（注：NodePort 通过 iptables 转发，无进程监听端口，此现象正常）

**验证过程**
1. 在 Master 节点上执行 `curl --interface 192.168.2.10 http://192.168.2.10:32074`，返回 HTTP 200 和 WordPress 页面 → 证明 NodePort 在 Master 上完全正常
2. 在 Windows 宿主机上执行 `Test-NetConnection 192.168.2.10 -Port 32074`，返回 `TcpTestSucceeded: True` → 证明 TCP 连接可达
3. 在 Windows 宿主机上执行 `curl --noproxy "*" http://192.168.2.10:32074`，返回 WordPress 页面 → 确认是代理拦截

**根因**
1. Windows 宿主机上开着代理软件（Clash / V2Ray 等），且绕过列表未包含 `192.168.2.0/23` 网段
2. 浏览器访问 `192.168.2.10:32074` 时，流量被代理软件截获并转发到远程代理服务器
3. 代理服务器无法路由内网 IP，导致连接超时
4. NodePort 本身完全正常，原记录中“NodePort 只在 Worker 节点监听”为错误推断，已被验证推翻

**解决方案**
```bash
# 方案一：在代理软件的绕过列表中添加内网 IP 段（推荐，一劳永逸）
# 在 Clash / V2Ray 等代理软件的 Bypass / No Proxy 列表中添加：
192.168.2.0/23

# 方案二：临时关闭系统代理，刷新浏览器访问
# 方案三：用 curl 绕过代理访问（仅用于测试验证）
curl --noproxy "*" http://192.168.2.10:32074

# 方案四：使用 kubectl port-forward（默认走 localhost，代理自动绕过）
kubectl port-forward -n wp-mysql-stack deployment/wp-web --address 0.0.0.0 9090:80
# 访问：http://localhost:9090 或 http://127.0.0.1:9090
```

**经验教训**
- `netstat` 看不到 NodePort 端口是**正常现象**（iptables 转发，无进程监听）
- 验证 NodePort 是否生效的正确方式：在节点上执行 `curl http://127.0.0.1:NodePort` 或 `curl --interface 节点IP http://节点IP:NodePort`
- 外部访问不通时，优先排查：宿主机代理设置、防火墙、虚拟机网络模式
- 代理软件会默认拦截所有 HTTP/HTTPS 流量，内网 IP 也需要显式加入绕过列表
- `Test-NetConnection` 和 `ping` 不受代理影响，能通不代表浏览器能通
- `kubectl port-forward` 监听的是 `localhost`，代理软件默认绕过本地回环地址，因此无需关闭代理即可使用
- 生产环境建议使用 Ingress 或 LoadBalancer 暴露服务，而非依赖 NodePort


### 6️⃣ 问题六：端口转发到 Service 失败

**现象**
- `kubectl port-forward -n wp-mysql-stack service/wp-web 9090:80` 显示 `Forwarding from ...`
- 但 `curl http://127.0.0.1:9090` 返回 `Connection refused`

**根因**
1. `port-forward` 转发到 Service 时需要经过 kube-proxy 和 Endpoints 进行流量转发
2. 某些环境因 Service 类型（NodePort）或网络插件配置问题导致转发链路不通

**解决方案**
```bash
# 直接转发到 Deployment（自动选择一个 Running 的 Pod），绕过 Service 层
kubectl port-forward -n wp-mysql-stack deployment/wp-web --address 0.0.0.0 9090:80
```

**经验教训**
- 转发到 Pod / Deployment 绕过了 Service 的 kube-proxy 层，连接更稳定可靠
- `deployment/xxx` 会自动选中一个 Running 的 Pod，无需手动查 Pod 名
- `port-forward` 到 Service 时依赖 kube-proxy 和 Endpoints 的正常工作，任一环节故障都会导致转发失败
- 开发调试时优先使用 `deployment/xxx` 进行端口转发，生产环境则建议使用 Ingress 或 LoadBalancer


### 7️⃣ 问题七：HPA 显示 `<unknown>`（缺少 CPU 请求）

**现象**
- `kubectl get hpa` 中 `TARGETS` 一直是 `<unknown>/50%`
- `kubectl top pods` 能正常显示 CPU 使用量
- HPA Events 报 `missing request for cpu in container wordpress`

**根因**
1. HPA 需要 Pod 容器定义 `resources.requests.cpu` 才能计算资源利用率
2. WordPress Deployment 中默认没有设置 `resources.requests`，导致 HPA 无法获取基准值

**解决方案**
```bash
# 为 WordPress 容器添加 CPU 和内存的请求与限制
kubectl patch deployment wp-web -n wp-mysql-stack --type='json' -p='[
  {"op": "add", "path": "/spec/template/spec/containers/0/resources", "value": {"requests": {"cpu": "100m", "memory": "128Mi"}, "limits": {"cpu": "200m", "memory": "256Mi"}}}
]'
```
等待 1-2 分钟，`TARGETS` 显示正常数值。

**经验教训**
- HPA 依赖 `resources.requests` 计算利用率，`kubectl top` 能看数据不代表 HPA 能用
- 必须为容器显式设置 CPU/内存请求（requests）和限制（limits），否则 HPA 无法工作
- 设置 requests 时需根据实际业务负载合理规划，过低会导致节点资源超卖，过高会造成资源浪费
- 可通过 `kubectl describe hpa` 查看 HPA 事件，快速定位 `<unknown>` 的具体原因




第二次登录：
```markdown
# 第二次登录：Kubernetes 集群搭建常见问题排查记录

本文档记录了在从零搭建 Kubernetes 集群过程中遇到的三个核心问题及其解决方案，涵盖节点资源管理、网络插件配置和容器镜像拉取三个方面。


## 问题一：Swap 分区未关闭

### 现象

- `kubectl get nodes` 显示节点状态为 `NotReady`
- `systemctl status kubelet` 报错：
  ```
  running with swap on is not supported, please disable swap
  ```

### 本质原因

Kubernetes 为了保证 Pod 运行的**确定性**和**稳定性**，要求**完全掌控节点的物理内存**。Linux 的 Swap（交换分区/文件）会让这种掌控失效。

### 为什么 Kubernetes 必须关闭 Swap？

| 问题 | 说明 |
|------|------|
| **调度决策被误导** | K8s 调度器根据 `requests.memory` 决定 Pod 调度位置。Swap 启用时，节点物理内存不足会触发内存交换，调度器看到的是“物理内存 + Swap”，误以为资源充足，实际数据已在磁盘中，性能急剧下降。 |
| **OOM 驱逐不准确** | 节点内存耗尽时 Linux 内核触发 OOM Killer。K8s 的 QoS 机制希望优先杀非核心 Pod，但 Swap 介入后，内核可能误杀核心业务 Pod。 |
| **内存限制失效** | `limits.memory` 通过 cgroups 限制 Pod 内存。Swap 允许 Pod 将内存数据交换到磁盘，绕过 cgroups 限制，造成“限而不死”的假象。 |

### 解决方案

在所有节点上执行以下命令：

```bash
# 临时关闭 Swap
sudo swapoff -a

# 永久禁用 Swap（注释 /etc/fstab 中的 swap 挂载项）
sudo sed -i '/swap/s/^/#/' /etc/fstab

# 验证是否已关闭
free -h | grep Swap
# 应显示 Swap: 0B 0B 0B
```

执行后重启 kubelet 验证：

```bash
sudo systemctl restart kubelet
kubectl get nodes
```


## 问题二：Flannel 健康检查探针失败（404）

### 现象

- `kubectl get pods -n kube-flannel` 显示 `0/1 Running`
- `kubectl describe pod` 报错：
  ```
  Readiness probe failed: HTTP probe failed with statuscode: 404
  ```

### 本质原因

Flannel 主容器实际已正常运行（日志显示 `bootstrap done`，网络配置已写入 `/run/flannel/subnet.env`），但 **Readiness Probe（就绪探针）配置的 HTTP 路径在新版本中已移除**，导致探针失败，Pod 被标记为“未就绪”。

### 为什么会出现这个问题？

这是典型的**配置版本脱节**问题：

- **第一次部署**：拉取的是 Flannel 旧版本（如 v0.24.x），该版本存在 `/readyz` 端点，探针通过。
- **第二次启动**：节点没有缓存旧镜像，kubelet 拉取了**最新版本**（如 v0.28.x）。新版本中开发者已移除 `/readyz` 端点，只保留 `/healthz`。
- 结果：YAML 配置文件仍指向 `/readyz`，新版镜像没有该路径，返回 404。

### 关键点

这只是一个**配置与镜像版本不一致导致的误报**，Flannel 的网络功能本身正常（Pod 能获得 IP，节点间通信正常）。

### 解决方案

将 `readinessProbe` 从 `httpGet` 改为 `tcpSocket`，只检查端口是否监听，不再依赖具体 HTTP 路径：

```bash
kubectl patch daemonset kube-flannel-ds -n kube-flannel --type='json' -p='[
  {
    "op": "replace",
    "path": "/spec/template/spec/containers/0/readinessProbe",
    "value": {
      "failureThreshold": 3,
      "tcpSocket": {
        "port": 8081
      },
      "initialDelaySeconds": 5,
      "periodSeconds": 10,
      "successThreshold": 1,
      "timeoutSeconds": 1
    }
  }
]'
```

> `tcpSocket` 方式检查 8081 端口是否可连接，只要端口在监听即视为 Pod 就绪，版本升级时更稳定。

验证修复：

```bash
kubectl get pods -n kube-flannel
# 应显示 1/1 Running
```


## 问题三：镜像拉取失败（ImagePullBackOff）

### 现象

- 测试 Pod 状态为 `ImagePullBackOff`
- `kubectl describe pod` 报错：
  ```
  Failed to pull image: proxyconnect tcp: dial tcp 192.168.2.1:10808: connect: connection refused
  ```

### 本质原因

节点系统环境变量设置了代理（`HTTP_PROXY` / `HTTPS_PROXY`），指向 `192.168.2.1:10808`，但该代理服务不可达。所有拉取 Docker Hub 镜像的请求都被“截胡”到代理，然后被拒绝。

### 为什么配置了镜像源“没用”？

系统级 `HTTP_PROXY` **优先级高于**容器运行时（containerd/docker）的镜像源配置。即使配置了阿里云镜像加速，所有流量也会先被代理拦截，导致镜像源配置失效。

### 解决方案

**临时方案**（让集群先跑起来）：

```bash
# 方式一：确保代理服务正常运行
# 或

# 方式二：临时移除系统代理（如果不需要代理）
unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY
# 从 /etc/environment 中删除代理相关行
```

**长期方案**（推荐，彻底解决）：

移除系统级代理，改为在容器运行时中配置国内镜像加速器。

#### 对于 containerd：

编辑 `/etc/containerd/config.toml`：

```toml
[plugins."io.containerd.grpc.v1.cri".registry.mirrors]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
    endpoint = ["https://docker.mirrors.ustc.edu.cn"]
    # 也可使用阿里云镜像加速（需注册获取个人加速地址）
    # endpoint = ["https://<你的ID>.mirror.aliyuncs.com"]
```

重启 containerd 和 kubelet：

```bash
sudo systemctl restart containerd
sudo systemctl restart kubelet
```

#### 对于 Docker：

编辑 `/etc/docker/daemon.json`：

```json
{
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"]
}
```

重启 Docker 和 kubelet：

```bash
sudo systemctl restart docker
sudo systemctl restart kubelet
```


## 附录：Flannel 是什么？

Flannel 是 Kubernetes 的 CNI（容器网络接口）网络插件，核心职责：

1. **为每个 Pod 分配全局唯一的 IP 地址**，避免 IP 冲突
2. **打通不同节点上 Pod 之间的网络通信**，通过 VXLAN 或 host-gw 等方式建立跨主机网络隧道

> 没有 Flannel，Node1 上的 `wp-web` Pod 和 Node2 上的 `wp-db` Pod 之间无法通信。通过 Service 名称访问数据库，底层依赖的就是 Flannel 提供的跨主机网络能力。


## 问题总结

| 问题 | 领域 | 本质 | 解决方案 |
|------|------|------|----------|
| **Swap 未关闭** | 节点资源管理 | K8s 要求绝对掌控内存，Swap 干扰调度和驱逐准确性 | `sudo swapoff -a` + 注释 fstab |
| **Flannel 探针 404** | 网络插件 | YAML 中的 `/readyz` 路径与新版本镜像不匹配 | 改为 `tcpSocket` 探针 |
| **ImagePullBackOff** | 容器运行时 | 系统代理拦截拉取请求，但代理不通 | 移除代理 + 配置国内镜像源 |

三个问题分别覆盖了**系统层**、**网络层**和**容器运行时层**，是 Kubernetes 集群从下到上核心链路的常见故障点。
```

