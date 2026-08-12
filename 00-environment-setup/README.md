一.
1. wsl -d Ubuntu-24.04
2.sudo sed -i 's/archive.ubuntu.com/mirrors.aliyun.com/g' /etc/apt/sources.list
3.sudo apt update
4.sudo apt install -y docker.io
二.安装minikube以及kubectl，Helm
1.sudo adduser w1（先在root给权限 usermod -aG sudo w1）
2.curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
3.sudo install minikube-linux-amd64 /usr/local/bin/minikube
4.curl -LO "https://dl.k8s.io/release/v1.28.2/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
5.curl -x http://本机地址:魔法监听(出站)端口 -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
现在检查所需环境：
1.minikube version
三.安装docker
sudo apt update
sudo apt install -y docker.io
sudo usermod -aG docker $USER
docker --version
四.启动minikube（魔法版）
1.先在win+r 输入wf.msc添加入站规则，加入你魔法端口，然后改Ubuntu代理
export http_proxy="http://本机地址:魔法监听(出战)端口"
export https_proxy="http://"

curl -I https://www.google.com测试生效与否
2.启动安装：
minikube start --driver=docker \
  --kubernetes-version=v1.28.2 \
  --docker-env HTTP_PROXY="http://本机地址:魔法监听(出站)端口" \
  --docker-env HTTPS_PROXY="http://本机地址:魔法监听(出站)端口" \
  --docker-env NO_PROXY="localhost,127.0.0.1,10.96.0.0/12,192.168.59.0/24,192.168.49.0/24,192.168.39.0/24"
3.验证：
unset http_proxy https_proxy
kubectl get nodes
