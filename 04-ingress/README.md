# 1. 启用 Ingress 控制器（Minikube 默认未启用）
minikube addons enable ingress

# 2. 查看 Ingress 控制器是否就绪
kubectl get pods -n ingress-nginx -w

# 3. 部署 Ingress
kubectl apply -f 05-ingress/nginx-ingress.yaml

# 4. 查看 Ingress
kubectl get ingress                  # 查看 Ingress 资源
kubectl describe ingress nginx-ingress  # 查看详情

# 5. 通过 curl 测试域名路由
curl -H "Host: nginx.local" http://$(minikube ip)

# 6. 如果返回 ConfigMap 的内容，说明 Ingress 生效

# 7. 修改本地 hosts 文件（让浏览器能用 nginx.local 访问）
# Windows: C:\Windows\System32\drivers\etc\hosts
# 添加一行: <minikube_ip> nginx.local

# 8. 浏览器访问 http://nginx.local

# 9. 查看 Ingress 日志（排障用）
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# 10. 清理
kubectl delete -f 05-ingress/nginx-ingress.yaml
minikube addons disable ingress      # 禁用 Ingress 控制器

Ingress 是七层路由，按域名/路径分发流量

需要 Ingress Controller 才生效（Minikube 需手动启用）

host 字段指定域名，path 指定路径，backend 指向 Service

生产环境通常结合 TLS/SSL 做 HTTPS 暴露