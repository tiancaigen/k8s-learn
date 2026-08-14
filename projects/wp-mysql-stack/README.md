# 🚀 基于 Kubeadm 的 Kubernetes 生产级 WordPress 部署实践

> 使用 Kubeadm 从零搭建多节点 Kubernetes 集群，完整实践 Deployment、StatefulSet、PV/PVC、Service、Ingress、ConfigMap、Secret 等 7 种核心资源，并实现 HPA 弹性伸缩、Helm 部署、持久化存储等企业级能力。


## 📋 项目概述

本项目基于 Kubernetes 集群，完整部署了一套高可用的 WordPress + MySQL 应用。通过手动搭建集群、配置存储、编排资源、配置网络、实现自动扩缩容等全流程实践，深入理解了云原生应用的生产级部署思路。

> **这是一个全面的 K8s 实战项目，涵盖从集群搭建到应用上线的完整链路。**


## 🎯 核心目标

- 从零到一搭建一个可投入开发测试的 Kubernetes 集群
- 部署 WordPress + MySQL，完成 7 种核心 K8s 资源的编排实战
- 实践 HPA 弹性伸缩、Helm 部署、持久化存储等扩展技能
- 验证云原生应用的持久化、弹性与可观测性


## 🛠️ 技术栈

| 领域 | 技术 |
|---|---|
| **集群搭建** | Kubeadm、Kubelet、Kubectl、Containerd |
| **网络插件** | Flannel（CNI） |
| **容器编排** | Deployment、StatefulSet、Service（ClusterIP / NodePort）、Ingress |
| **配置管理** | ConfigMap、Secret |
| **持久化存储** | PV / PVC |
| **弹性伸缩** | HPA（Horizontal Pod Autoscaler） |
| **包管理工具** | Helm（Bitnami Charts） |
| **应用栈** | WordPress（Deployment）、MySQL（StatefulSet） |
| **操作系统** | Ubuntu 22.04 / 24.04 |


## ✨ 项目亮点

### 7 种核心 K8s 资源全覆盖

| 资源类型 | 用途 | 实战内容 |
|---|---|---|
| **Deployment** | WordPress 无状态应用部署 | 2 副本 + 滚动更新 |
| **StatefulSet** | MySQL 有状态数据库部署 | 1 副本 + 稳定的网络标识 |
| **Service (ClusterIP)** | 集群内部服务发现 | MySQL 服务暴露给 WordPress |
| **Service (NodePort)** | 外部访问入口 | 宿主机端口映射访问 WordPress |
| **PV / PVC** | 持久化存储 | MySQL 数据持久化 |
| **ConfigMap** | 配置注入 | WordPress 环境变量配置 |
| **Secret** | 敏感信息管理 | MySQL 数据库密码加密存储 |

### 扩展技能

- **HPA 弹性伸缩**：基于 CPU 使用率自动扩缩容 WordPress Pod
- **Helm 部署**：使用 Bitnami Charts 一键部署 WordPress + MariaDB
- **持久化存储验证**：数据库 Pod 重启后数据不丢失
- **port-forward 调试**：绕过 NodePort 网络策略，快速本地访问验证
- **kubeadm 全手动搭建**：完整记录环境排障（7 个典型问题）


## 🔧 环境排障记录

部署过程中遇到并解决的典型问题：

| 问题 | 根因 | 解决方案 |
|---|---|---|
| `kubeadm init` 超时 | pause 镜像被墙 | 手动拉取阿里云镜像并打标签 |
| Flannel Pod CrashLoopBackOff | ghcr.io 被墙 + install-conf 缺失 | 使用 DaoCloud 镜像源 + patch DaemonSet |
| MySQL Pod ImagePullBackOff | 镜像拉到 Worker 节点 | 在对应节点 `ctr image import` |
| WordPress 连接 MySQL 失败 | 数据库未自动创建 | `kubectl exec` 手动 `CREATE DATABASE` |
| Windows 无法访问 NodePort | 代理软件拦截内网 IP | 绕过列表添加 `192.168.2.0/23` |


## 🚀 后续优化方向

- [ ] **Ingress 域名访问**：部署 Ingress Controller，配置域名路由
- [ ] **监控告警**：部署 Prometheus + Grafana 监控集群和应用指标
- [ ] **日志收集**：ELK / Loki 日志采集与分析
- [ ] **CI/CD 集成**：GitLab CI / GitHub Actions 自动构建镜像并部署
- [ ] **安全加固**：RBAC 权限控制、Pod Security Policy、Network Policy
- [ ] **私有镜像仓库**：部署 Harbor，实现镜像统一管理


## 📝 经验总结

1. **环境准备是成败关键**：Swap 关闭、内核模块加载、cgroup 驱动一致性，缺一不可
2. **国内部署优先使用镜像源**：`registry.aliyuncs.com/google_containers` 和 `m.daocloud.io` 可有效避免拉取超时
3. **调试工具链要熟练**：`kubectl describe`、`kubectl logs`、`kubectl exec`、`crictl`、`journalctl` 是排障的瑞士军刀
4. **NodePort 访问踩坑**：外部访问不通优先排查代理设置，而非直接归因于 K8s 配置
5. **port-forward 是调试神器**：绕过网络策略，快速验证应用是否正常
6. **HPA 依赖 resources.requests**：不设置 requests，HPA 永远显示 `<unknown>`


## 🙏 致谢

感谢开源社区和 Bitnami 提供的优质 Helm Charts，为本项目的快速验证提供了便利。


## 📄 License

本项目仅供学习交流使用，遵循 MIT License。