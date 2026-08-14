# Kubernetes 学习路线图

从零开始通过 kubeadm 搭建生产级集群，逐步深入核心资源、存储、监控、安全、CI/CD 及服务网格等方向，目标是具备独立运维 K8s 集群的能力。


## 🗺️ 学习计划大纲

| 阶段 | 基础内容 | 进阶扩展（⭐） |
|---|---|---|
| **一、核心基础** | Pod / Deployment / Service / Ingress / ConfigMap / Secret | 金丝雀发布 / 自定义 Helm Chart / CSI 存储 |
| **二、生产集群搭建** | kubeadm 单 Master 集群部署（核心项目） | 3 Master + HAProxy + Keepalived 高可用 |
| **三、应用部署与存储** | StatefulSet / PV / PVC / StorageClass | NFS CSI / Rook Ceph / Velero 备份 |
| **四、可观测性** | Prometheus / Grafana / Alertmanager | Loki 日志 / Jaeger 链路追踪 |
| **五、安全与权限** | RBAC / ServiceAccount | NetworkPolicy / PSA / OPA Gatekeeper |
| **六、CI/CD** | Jenkins Pipeline | ArgoCD GitOps |
| **七、云原生扩展** | - | Istio / VPA / Cluster Autoscaler / Knative / Operator / Karmada |


## 📂 目录结构

```
k8s-learning/
├── 00-environment-setup/                    # 基础：Minikube / kind 本地环境
├── 01-pod/                                  # 基础：Pod 生命周期、多容器模式
├── 02-deployment/                           # 基础：Deployment 滚动更新与回滚
├── 03-service/                              # 基础：ClusterIP / NodePort / LoadBalancer
├── 03.5-network/                            # ⭐ 进阶：CNI 网络插件（Calico / Cilium）
│   ├── calico-ipip-bgp/                     # 网络策略与 BGP 路由反射
│   ├── networkpolicy/                       # 细粒度网络隔离实战
│   └── troubleshooting/                     # 容器内抓包与网络排错
├── 04-ingress/                              # 基础：Ingress Controller（Nginx Ingress）
│   ├── gateway-api-up/                      # ⭐ Gateway API（K8s 官方下一代 Ingress）
│   └── traefik-up/                          # ⭐ Traefik 作为 Ingress Controller
├── 05-configmap-secret/                     # 基础：配置与敏感数据管理
├── 06-storage/                              # 基础：PV / PVC / StorageClass
│   └── csi-up/                              # ⭐ NFS CSI / Rook Ceph 对接
├── 06.5-disaster-recovery-up/               # ⭐ Velero 集群备份与迁移
├── 07-helm/                                 # 基础：Helm 安装与常用 Chart 部署
│   └── custom-chart-up/                     # ⭐ 编写并发布自己的 Chart
├── 08-observability/                        # 基础：Prometheus + Grafana 监控
│   ├── prometheus-operator-up/              # ⭐ ServiceMonitor 动态发现
│   ├── alertmanager-up/                     # ⭐ 告警分组/抑制/静默治理
│   ├── logging-up/                          # ⭐ 集成 Loki 轻量日志
│   └── tracing-up/                          # ⭐ Jaeger / OpenTelemetry 链路追踪
├── 09-security/                             # 基础：RBAC（ServiceAccount / Role / ClusterRole）
│   ├── networkpolicy-up/                    # ⭐ NetworkPolicy 零信任实践
│   ├── psa-up/                              # ⭐ Pod Security Admission
│   └── opa-gatekeeper-up/                   # ⭐ OPA 策略引擎（准入控制）
├── 10-service-mesh-up/                      # ⭐ Istio / Linkerd 服务网格
│   ├── traffic-management/                  # 灰度发布、故障注入
│   └── observability/                       # Kiali 可视化 + Jaeger 链路追踪
├── 11-k8s-advanced-up/                      # ⭐ HPA / CronJob / DaemonSet / Job
│   └── auto-scaling/                        # VPA / Cluster Autoscaler 集群弹性
├── 12-k8s-ha-up/                            # ⭐ 核心项目：kubeadm 生产级高可用集群
│   ├── ha-topology/                         # 多 Master + etcd 集群架构
│   ├── loadbalancer/                        # 控制平面负载均衡（Keepalived / HAProxy）
│   └── backup/                              # etcd 定期备份与恢复演练
├── 13-ci-cd-integration-up/                 # ⭐ Jenkins Pipeline / GitHub Actions
│   ├── gitops/                              # ArgoCD 持续交付 + GitOps 工作流
│   └── argo-rollouts/                       # Argo Rollouts 灰度发布
├── 14-logging-up/                           # ⭐ EFK / ELK 日志平台搭建
│   └── loki-stack/                          # Loki + Promtail 轻量日志方案
├── 15-cloud-native-up/                      # ⭐ 云原生扩展
│   ├── operator-pattern/                    # Operator SDK 开发入门
│   ├── multicluster/                        # Karmada / Kubefed 多集群联邦管理
│   ├── serverless-knative/                  # Knative Serverless 工作负载
│   └── ai-bigdata/                          # Kubeflow / Ray / Spark on K8s
├── scripts/                                 # 辅助工具脚本
└── notes/                                   # 踩坑记录 & 面试高频问答
    ├── troubleshooting/                     # 常见故障现象与修复步骤
    └── interview-qa/                        # 面试常考知识点梳理
```




## 🛠️ 技术栈

### ✅ 基础

| 类别 | 技术 | 用途 |
|---|---|---|
| **编排** | Kubernetes v1.28.2 | 容器编排调度 |
| **部署工具** | kubeadm | 生产级集群部署 |
| **本地环境** | Minikube / kind | 本地开发测试 |
| **容器运行时** | Docker / containerd | 容器运行环境 |
| **命令行工具** | kubectl | K8s 集群管理 |
| **服务暴露** | Nginx Ingress Controller | 七层流量路由 |
| **服务发现** | CoreDNS | 集群内部 DNS 解析 |
| **包管理** | Helm | K8s 应用包管理 |
| **监控** | Prometheus + Grafana + Alertmanager | 指标采集、可视化、告警 |
| **权限** | RBAC | 权限控制与最小权限原则 |
| **CI/CD** | Jenkins | 自动化构建与部署 |


### ⭐ 进阶

| 类别 | 技术 | 用途 |
|---|---|---|
| **网络** | Calico / Cilium | CNI 网络插件与 NetworkPolicy |
| **API 网关** | Gateway API / Traefik | 下一代 Ingress 与七层代理 |
| **服务网格** | Istio / Linkerd | 灰度发布、流量管理、可观测性 |
| **存储** | Rook Ceph / NFS CSI | 云原生分布式存储与动态供给 |
| **备份** | Velero | 集群备份与迁移 |
| **日志** | Loki / EFK | 轻量日志聚合与集中管理 |
| **链路追踪** | Jaeger / OpenTelemetry | 分布式调用链追踪 |
| **安全** | OPA Gatekeeper / PSA | 策略引擎与 Pod 安全标准 |
| **GitOps** | ArgoCD | 声明式持续交付 |
| **灰度发布** | Argo Rollouts | 渐进式交付与金丝雀发布 |
| **自动伸缩** | VPA / Cluster Autoscaler | 垂直伸缩与集群节点自动伸缩 |
| **Serverless** | Knative | 云原生 Serverless 工作负载 |
| **多集群管理** | Karmada | 跨集群统一调度与联邦管理 |
| **AI/大数据** | Kubeflow / Ray / Spark on K8s | 云原生 AI 与大数据平台 |
| **辅助工具** | kubectx / k9s / yq | 集群切换、终端管理、YAML 处理 |



。。。。。。。。。。。。。。。。。。。。。。。。。。。


## 🧩 环境排障记录

### 1️⃣ 问题一：kubeadm init 超时 — pause 镜像拉取失败

**现象**
- `kubeadm init` 卡在 `Waiting for a healthy API server`，4 分钟后超时退出
- 报错：`context deadline exceeded`、`dial tcp ... connection refused`
- `sudo crictl ps -a` 无任何控制平面容器

**根因**
- `registry.k8s.io/pause:3.10.1` 镜像无法拉取
- containerd 的 `sandbox_image` 配置虽改为阿里云地址，但未生效
- kubelet 的 `--pod-infra-container-image` 参数优先级更高，仍指向官方源

**解决方案**
```bash
# 手动拉取阿里云 pause 镜像并打上官方标签
sudo crictl pull registry.aliyuncs.com/google_containers/pause:3.10
sudo ctr -n k8s.io image tag registry.aliyuncs.com/google_containers/pause:3.10 registry.k8s.io/pause:3.10.1

# 重置并重新初始化
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes /var/lib/etcd
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --image-repository=registry.aliyuncs.com/google_containers

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
