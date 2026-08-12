# Kubernetes 学习仓库

从零开始通过 Minikube 在 WSL2 + Ubuntu 上搭建 Kubernetes 学习环境，记录各核心模块的实验过程与操作笔记。

## 环境信息

| 项目 | 版本 |
|---|---|
| 操作系统 | WSL2 + Ubuntu 24.04 |
| 容器运行时 | Docker |
| Kubernetes | v1.28.2 |
| Minikube | v1.38.1 |
| IDE | VS Code + Remote WSL |

## 完整目录结构

k8s-learning/
├── 00-environment-setup/          # 环境搭建（WSL2 + Minikube + Docker）
│   ├── minikube-start.sh
│   ├── docker-install.sh
│   └── proxy-switch.sh
│
├── 01-pod/                        # Pod 基础实验
│   ├── nginx-pod.yaml
│   ├── multi-container-pod.yaml
│   └── pod-lifecycle-demo.yaml    # Pod 生命周期（Pending → Running → Terminating）
│
├── 02-deployment/                 # Deployment 控制器（滚动更新/回滚/扩缩容）
│   ├── nginx-deployment.yaml
│   ├── rolling-update-demo.yaml
│   └── rollback-demo.yaml
│
├── 03-service/                    # Service 服务暴露（ClusterIP/NodePort/LoadBalancer）
│   ├── nginx-service-clusterip.yaml
│   ├── nginx-service-nodeport.yaml
│   └── nginx-service-loadbalancer.yaml
│
├── 04-ingress/                    # Ingress 七层路由（域名 + 路径分发）
│   ├── nginx-ingress.yaml
│   └── ingress-nginx-values.yaml
│
├── 05-configmap-secret/           # 配置管理（ConfigMap + Secret）
│   ├── nginx-configmap.yaml
│   ├── mysql-secret.yaml
│   └── deployment-with-configmap.yaml
│
├── 06-storage/                    # 持久化存储（PV/PVC/StorageClass/StatefulSet）
│   ├── pv-nfs.yaml
│   ├── pvc.yaml
│   ├── storage-class.yaml
│   └── mysql-statefulset.yaml
│
├── 07-helm/                       # Helm 包管理
│   └── my-app/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           └── service.yaml
│
├── 08-observability/              # 监控与可观测性（Prometheus + Grafana + Alertmanager）
│   ├── prometheus-values.yaml
│   ├── grafana-values.yaml
│   ├── alertmanager-config.yaml
│   └── service-monitor.yaml
│
├── 09-security/                   # 安全（RBAC + NetworkPolicy + ServiceAccount）
│   ├── rbac-role.yaml
│   ├── rbac-rolebinding.yaml
│   ├── service-account.yaml
│   └── network-policy.yaml
│
├── 10-service-mesh/               # 服务网格（Istio 入门）
│   └── istio-demo.yaml
│
├── 11-k8s-advanced/               #  K8s 进阶（企业核心）
│   ├── hpa.yaml                   # Horizontal Pod Autoscaler 自动扩缩容
│   ├── cronjob.yaml               # CronJob 定时任务
│   ├── daemonset.yaml             # DaemonSet 每个节点一个 Pod（日志/监控）
│   ├── job.yaml                   # Job 一次性任务
│   └── poddisruptionbudget.yaml   # PDB 主动中断预算
│
├── 12-k8s-ha/                     # K8s 高可用与集群管理
│   ├── kubeadm-init.md            # kubeadm 初始化集群记录
│   ├── master-ha.md               # 高可用控制平面搭建思路
│   ├── etcd-backup.md             # etcd 备份与恢复
│   └── cluster-upgrade.md         # 集群升级步骤
│
├── 13-ci-cd-integration/          # CI/CD 集成与 GitOps
│   ├── jenkins-pipeline.md        # Jenkins Pipeline 与 K8s 集成
│   ├── github-actions.yaml        # GitHub Actions 部署到 K8s
│   ├── argocd-demo.yaml           # ArgoCD GitOps 应用
│   └── webhook-trigger.md         # Webhook 触发自动部署
│
├── 14-logging/                    # 日志收集（EFK/ELK）
│   ├── elasticsearch-values.yaml
│   ├── filebeat-daemonset.yaml
│   └── kibana-values.yaml
│
├── 15-cloud-native/               # 云原生生态扩展
│   ├── serverless.md              # Knative / OpenFaaS 入门
│   ├── operator-pattern.md        # Operator 模式理解
│   └── service-catalog.md         # Service Catalog 服务目录
│
├── scripts/                       # 辅助脚本
│   ├── deploy-all.sh
│   └── clean-all.sh
│
└── notes/                         # 笔记与踩坑记录
    ├── troubleshooting.md
    ├── k8s-cheatsheet.md
    └── interview-qa.md            # 面试问答整理



## 🛠️ 技术栈

| 类别 | 技术 | 版本 | 用途 |
|---|---|---|---|
| **核心编排** | Kubernetes | v1.28.2 | 容器编排调度 |
| | Minikube | v1.38.1 | 本地 K8s 开发集群 |
| | Docker | 29.1.3 | 容器运行时 |
| | kubectl | v1.28.2 | K8s 命令行管理工具 |
| **存储与网络** | Helm | - | K8s 应用包管理 |
| | Ceph / NFS | - | 持久化存储后端 |
| | Ingress-Nginx | - | 七层流量路由 |
| **可观测性** | Prometheus | - | 指标采集与监控告警 |
| | Grafana | - | 监控数据可视化 |
| | Alertmanager | - | 告警分组与抑制 |
| | EFK/ELK | - | 日志收集与分析 |
| **安全** | RBAC | - | 权限控制 |
| | NetworkPolicy | - | 网络策略 |
| | ServiceAccount | - | Pod 身份管理 |
| **CI/CD & GitOps** | Jenkins | - | CI/CD 流水线 |
| | GitHub Actions | - | 自动化构建与部署 |
| | ArgoCD | - | GitOps 声明式持续部署 |
| **云原生扩展** | Istio | - | 服务网格（流量管理/可观测性） |
| | Knative | - | Serverless 工作负载 |
| | Operator | - | K8s 应用自动化管理 |
| **辅助工具** | kubectx / kubens | - | 快速切换集群与命名空间 |
| | k9s | - | 终端 K8s 集群管理 |
| | yq | - | YAML 处理工具 |


