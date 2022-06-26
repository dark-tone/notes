# 关闭swap
```
// 临时关闭
swapoff -a
// 永久关闭
sed -ri 's/.*swap.*/#&/' /etc/fstab
```

# 更改host
```
// 临时修改hostname
hostname xxxx

// 永久修改hostname
vim /etc/hostname 更改所需的hostname

// 更改hosts映射关系
vim /etc/hosts

// 重启应用配置
reboot
```




# kubeadm、kubectl、kubelet
## 添加阿里镜像源
```
cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb http://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -

apt-get update
```

## 下载kubeadm所需的镜像
```
// 列出所需的镜像及对应版本
kubeadm config images list

// 下载及打标签（注意版本）
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.24.0
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.24.0
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.24.0
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.24.0
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.7
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.5.3-0
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.8.6

docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.24.0 k8s.gcr.io/kube-apiserver:v1.24.0
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.24.0 k8s.gcr.io/kube-controller-manager:v1.24.0
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.24.0 k8s.gcr.io/kube-scheduler:v1.24.0
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.24.0 k8s.gcr.io/kube-proxy:v1.24.0
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.7 k8s.gcr.io/pause:3.7
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.5.3-0 k8s.gcr.io/etcd:3.5.3-0
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.8.6 k8s.gcr.io/coredns/coredns:v1.8.6

docker image rm registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.24.0
docker image rm registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.24.0
docker image rm registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.24.0
docker image rm registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.24.0
docker image rm registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.7
docker image rm registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.5.3-0
docker image rm registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.8.6
```

## 初始化
```
// 高级版本可用--image-repository指定镜像地址，这样做可省略打镜像tag的步骤
kubeadm init xxx
```

## 其他
kubeadm init发生错误时，可用kubeadm reset重新初始化


kubeadm token create --print-join-command 重新生成join命令


配置kubectl工具(v1.21.12)
```
mkdir -p /root/.kube
cp /etc/kubernetes/admin.conf /root/.kube/config
```

配置网络（flannel）
```
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

kubectl apply -f kube-flannel.yml
```


部署nginx尝试
```
# 部署nginx
kubectl create deployment nginx --image=nginx
# 暴露端口
kubectl expose deployment nginx --port=80 --type=NodePort
```