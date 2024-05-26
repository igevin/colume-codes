
# 3. RKE 部署 Kubernetes 集群
## 3.1 准备工作


### 3.1.2 修改 /etc/sysctl.conf

```shell
# vim /etc/sysctl.conf
cat > /etc/sysctl.conf << EFO
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EFO

# 生效
sysctl -p /etc/sysctl.conf
```

### 3.1.3 关闭防火墙、swap

```shell
# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭selinux
sed -i 's/enforcing/disabled/' /etc/selinux/config  # 永久
setenforce 0  # 临时

## 关闭swap
swapoff -a && sed -ri 's/.*swap.*/#&/' /etc/fstab
```

### 3.1.4 创建用户

```shell
# 创新 rancher 用户
useradd rancher

# 为 rancher 用户设置密码
passwd rancher

# 确保 rancher 用户能使用 docker
usermod -aG docker rancher
```


### 3.1.5 RKE 所在主机上（本文即 rke1）创建密钥，用于部署集群

```shell
ssh-keygen

# 将所生成的密钥的公钥分发到各个节点
ssh-copy-id rancher@rke1
ssh-copy-id rancher@rke2
ssh-copy-id rancher@rke3
```


# 4. 如何在内网环境中部署 Kubernetes 集群

```shell
#!/bin/bash

images=(
    "rancher/mirrored-coreos-etcd:v3.5.10"
    "rancher/rke-tools:v0.1.96"
    "rancher/rke-tools:v0.1.96"
    "rancher/rke-tools:v0.1.96"
    "rancher/rke-tools:v0.1.96"
    "rancher/mirrored-k8s-dns-kube-dns:1.22.28"
    "rancher/mirrored-k8s-dns-dnsmasq-nanny:1.22.28"
    "rancher/mirrored-k8s-dns-sidecar:1.22.28"
    "rancher/mirrored-cluster-proportional-autoscaler:1.8.6"
    "rancher/mirrored-coredns-coredns:1.10.1"
    "rancher/mirrored-cluster-proportional-autoscaler:1.8.6"
    "rancher/mirrored-k8s-dns-node-cache:1.22.28"
    "rancher/hyperkube:v1.27.11-rancher1"
    "rancher/mirrored-flannel-flannel:v0.21.4"
    "rancher/flannel-cni:v0.3.0-rancher8"
    "rancher/mirrored-calico-node:v3.26.3"
    "rancher/calico-cni:v3.26.3-rancher1"
    "rancher/mirrored-calico-kube-controllers:v3.26.3"
    "rancher/mirrored-calico-ctl:v3.26.3"
    "rancher/mirrored-calico-pod2daemon-flexvol:v3.26.3"
    "rancher/mirrored-calico-node:v3.26.3"
    "rancher/calico-cni:v3.26.3-rancher1"
    "rancher/mirrored-calico-kube-controllers:v3.26.3"
    "rancher/mirrored-flannel-flannel:v0.21.4"
    "rancher/mirrored-calico-pod2daemon-flexvol:v3.26.3"
    "weaveworks/weave-kube:2.8.1"
    "weaveworks/weave-npc:2.8.1"
    "rancher/mirrored-pause:3.7"
    "rancher/nginx-ingress-controller:nginx-1.9.4-rancher1"
    "rancher/mirrored-nginx-ingress-controller-defaultbackend:1.5-rancher1"
    "rancher/mirrored-ingress-nginx-kube-webhook-certgen:v20231011-8b53cabe0"
    "rancher/mirrored-metrics-server:v0.6.3"
    "rancher/mirrored-pause:3.7"
    "noiro/cnideploy:6.0.4.1.81c2369"
    "noiro/aci-containers-host:6.0.4.1.81c2369"
    "noiro/opflex:6.0.4.1.81c2369"
    "noiro/opflex:6.0.4.1.81c2369"
    "noiro/openvswitch:6.0.4.1.81c2369"
    "noiro/aci-containers-controller:6.0.4.1.81c2369"
)

# 新的仓库地址
new_registry="xxx.xxx.com/xxx"

# 遍历镜像列表
for image in "${images[@]}"; do
    echo "原始镜像：$image"
    # 拉取原始镜像
    docker pull "$image"

    # 确定原始镜像的名称和标签
    name=$(echo "$image" | awk -F'[:/]' '{print $2}')
    tag=$(echo "$image" | awk -F':' '{print $2}')
    echo $name $tag

    # 构建新的镜像标签
    new_image="${new_registry}/${name}:${tag}"

    # 为下载的镜像打上新标签
    # docker tag "$image" "$new_image"

    # 输出结果
    echo "原镜像 $image 已重新标记为 $new_image"


    echo "push $new_image"
    # docker push $new_image

done

echo "所有镜像处理完成。"

```

# 5. 使用集群

```shell
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"

chmod +x kubectl
mv kubectl /usr/local/bin/

kubectl version --client
Client Version: v1.29.2
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3

mkdir ~/.kube
cp kube_config_cluster.yml /root/.kube/
mv /root/.kube/kube_config_cluster.yml /root/.kube/config
```

```shell
docker run -d --restart=unless-stopped --privileged --name rancher -p 80:80 -p 443:443 rancher/rancher:v2.8
```