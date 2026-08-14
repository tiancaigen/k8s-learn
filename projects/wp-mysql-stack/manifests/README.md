# ============================================================
# WordPress + MySQL 完整部署 YAML 资源清单
# 包含：Namespace、Secret、PV/PVC、MySQL StatefulSet、
# WordPress Deployment + ConfigMap + Service (NodePort)、
# Ingress（可选）、HPA（可选）
# 
# 使用方式：
#   # 直接应用全部资源（建议按顺序单个应用）
#   kubectl apply -f 00-namespace.yaml
#   kubectl apply -f 01-secret.yaml
#   kubectl apply -f 02-pv-pvc.yaml
#   kubectl apply -f 03-mysql.yaml
#   kubectl apply -f 04-wordpress.yaml
#   kubectl apply -f 05-ingress.yaml   # 可选
#   kubectl apply -f 06-hpa.yaml       # 可选
# 
#   # 或合并为单个文件后整体应用
#   cat *.yaml > all-in-one.yaml
#   kubectl apply -f all-in-one.yaml
# 
# 清理：
#   kubectl delete namespace wp-mysql-stack
# ============================================================

# 00-namespace.yaml
# 创建独立命名空间 wp-mysql-stack，用于隔离所有资源。
---
apiVersion: v1
kind: Namespace
metadata:
  name: wp-mysql-stack
---
---
# 01-secret.yaml
# 存储 MySQL root 密码，Base64 编码（MTIzNDU2 对应明文 123456）。
---
apiVersion: v1
kind: Secret
metadata:
  name: wp-db-secret
  namespace: wp-mysql-stack
type: Opaque
data:
  root-password: MTIzNDU2
---
---
# 02-pv-pvc.yaml
# 持久化存储：PV 提供 1Gi 存储（HostPath 模拟），PVC 申请该存储。
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: wp-db-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/mysql
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wp-db-pvc
  namespace: wp-mysql-stack
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
---
# 03-mysql.yaml
# MySQL StatefulSet + Headless Service，使用 PVC 持久化数据，密码从 Secret 读取。
---
apiVersion: v1
kind: Service
metadata:
  name: wp-db
  namespace: wp-mysql-stack
spec:
  selector:
    app: wp-db
  ports:
    - port: 3306
  clusterIP: None
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: wp-db
  namespace: wp-mysql-stack
spec:
  serviceName: wp-db
  replicas: 1
  selector:
    matchLabels:
      app: wp-db
  template:
    metadata:
      labels:
        app: wp-db
    spec:
      containers:
        - name: mysql
          image: docker.io/library/mysql:5.7
          imagePullPolicy: Never
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: wp-db-secret
                  key: root-password
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: data
              mountPath: /var/lib/mysql
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: wp-db-pvc
---
---
# 04-wordpress.yaml
# WordPress Deployment（2 副本）+ ConfigMap（数据库连接配置）+ NodePort Service（外部访问）。
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: wp-config
  namespace: wp-mysql-stack
data:
  WORDPRESS_DB_HOST: wp-db
  WORDPRESS_DB_USER: root
  WORDPRESS_DB_NAME: wordpress
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wp-web
  namespace: wp-mysql-stack
spec:
  replicas: 2
  selector:
    matchLabels:
      app: wp-web
  template:
    metadata:
      labels:
        app: wp-web
    spec:
      containers:
        - name: wordpress
          image: wordpress:latest
          ports:
            - containerPort: 80
          env:
            - name: WORDPRESS_DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: wp-config
                  key: WORDPRESS_DB_HOST
            - name: WORDPRESS_DB_USER
              valueFrom:
                configMapKeyRef:
                  name: wp-config
                  key: WORDPRESS_DB_USER
            - name: WORDPRESS_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: wp-db-secret
                  key: root-password
            - name: WORDPRESS_DB_NAME
              valueFrom:
                configMapKeyRef:
                  name: wp-config
                  key: WORDPRESS_DB_NAME
---
apiVersion: v1
kind: Service
metadata:
  name: wp-web
  namespace: wp-mysql-stack
spec:
  type: NodePort
  selector:
    app: wp-web
  ports:
    - port: 80
      targetPort: 80
---
---
# 05-ingress.yaml（可选）
# 通过 Ingress 实现域名 wordpress.local 访问，需提前部署 Ingress Controller。
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wp-ingress
  namespace: wp-mysql-stack
spec:
  ingressClassName: nginx
  rules:
    - host: wordpress.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: wp-web
                port:
                  number: 80
---
---
# 06-hpa.yaml（可选）
# HPA 弹性伸缩：基于 CPU 使用率（50%）自动调整副本数（2~5 个）。
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: wp-web-hpa
  namespace: wp-mysql-stack
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: wp-web
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
---