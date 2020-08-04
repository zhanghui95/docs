
# Docker

## 1. Docker三个基本概念
 * 镜像(image)
 * 容器(container)
 * 仓库(repository)
 <font color='red'>理解了三个概念，就理解了Docker的整个生命周期！</font>

### 1.1 镜像
    1. 镜像：一个只读的模板，可以创建容器
    2. 镜像与容器的关系，类似于class类与对象之间的关系

### 1.2 容器
    1. docker利用容器来运行应用
    2. 容器是从镜像创建的运行实例，它可以被启动 开始 停止 删除
    3. 每个容器是相互隔离的，保证安全

### 1.3 Registry(仓库注册服务器)
    1. Registry是集中存放镜像文件的场所
    2. 仓库是对其中的镜像进行分类管理的
    3. 一个registry中会有多个repository
    4. 一个repository中会有多个不同tag的image

## 2. Docker镜像基本操作(CentOS7环境下)
    1. 查看是否安装过docker：yum list installed | grep docker
    2. 删除：yum remove -y 名称
    3. 安装：yum install docker
    4. 启动：systemctl start docker/systemctl start docker.service
    5. 查看状态：systemctl status docker
    6. 停止：systemctl stop docker
    7. 官方docker hub拉取镜像：docker pull mysql:5.5
    8. 从ustc上拉取：中国科学技术大学国内镜像
     宿主机编辑文件：vi /etc/docker/daemon.json中添加，没有此文件先创建
     {
         "registry-mirrors":["https://docker.mirrors.ustc.edu.cn"]
     }
     重启docker服务：systemctl restart docker.service
    9. 列出镜像：docker images
    10. 删除镜像：docker rmi 镜像名:版本号/docker rmi 镜像id(一个仓库只存在一个镜像的情况)
    11. 导入镜像：docker load < /root/centos7.tar.gz
    12. 导出镜像：docker save tomcat:7 > /root/tomcat7.tar.gz

## 3. Docker容器操作
 ### 3.1 启动容器
    1. 交互式启动：docker run -it --name 容器名称 镜像 /bin/bash
     eg：docker run it -name my-centos centos:7 /bin/bash
    2. 守护进程启动：docker run -d --name 容器名称 镜像
     查看已运行的容器：docker ps(正在运行)/docker ps -a(历史运行过)

 ### 3.2 停止
    docker stop 容器名称 eg：docker stop my-centos

 ### 3.3 重启
    docker start 容器名称或容器id

 ### 3.4 删除容器
    1. 删除指定容器：docker rm 容器名称或容器id
    2. 删除所有容器：docker rm 'docker ps -a -q'

## 4. 搭建Tomcat服务
    1. 端口映射 8888宿主机器的端口
     docker run -d --name my-tomcat -p 8888:8080 tomcat:7 
    2. 将部署的war放到docker中的tomcat里
     docker cp test.war my-tomcat:/usr/local/tomcat/webapps
    3. 进到容器里面
     docker exec -it my-tomcat /bin/bash
     ls
     cd webapps

# Kubernetes

## 1. 配置并安装k8s国内源

1.1 创建配置文件`sudo touch /etc/apt/sources.list.d/kubernetes.list`

1.2 添加写权限`sudo chmod 666 /etc/apt/sources.list.d/kubernetes.list`

再添加`deb http://mirrors.ustc.edu.cn/kubernetes/apt kubernetes-xenial main`

1.3 执行`sudo apt update`认证报错

1.4 添加认证key 错误信息NO_PUBKEY后8位

生成证书`sudo gpg --keyserver keyserver.ubuntu.com --recv-keys xxxxxxxx`

`sudo gpg --export --armor xxxxxxxx | sudo apt-key add -`

再次更新源`sudo apt update`

## 2. 禁止基础设施

2.1 禁止防火墙

```shell
sudo ufw disable
```

2.2 关闭swap

```shell
临时关闭：sudo swapoff -a
永久关闭：sudo vim /etc/fstab 注释swap行
```

2.3 禁止selinux(ubuntu自带安全)

```
sudo apt install -y selinux-utils
setenforce 0
shutdown -r now
sudo gentenfirce
```

## 3. k8s系统网络配置

3.1 配置内核参数 将桥接的IPv4流量传递到iptables的链

创建/etc/sysctl.d/k8s.conf文件

添加如下内容：

```ini
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
vm.swappiness = 0
```

让其生效：

```shell
sudo modprobe br_netfilter
sudo sysctl -p /etc/sysctl.d/k8s.conf
```

## 4. 安装k8s

> 切换到root用户

4.1 安装目标版本

```she
apt update && apt-get install -y kubelet=1.13.1-00 kubernetes-cni=0.6.0-00 kubeadm=1.13.1-00 kubectl=1.13.1-00
```

4.2 设置开机重启

```she
sudo systemctl enable kubelet && systemctl start kubelet
sudo shutdown -r now
```

## 5. 集群环境

5.1 配置分别配置 master及其他节点 主机名 ip hosts

5.2 配置master节点

创建工作目录

```shell
mkdir /home/stefan/working
cd /home/stefan/worling/
```

创建kubeadm.conf配置文件

```
创建k8s的管理工具hubeadm对应的配置文件，候选操作在/working/目录下
使用kubeadm配置文件，通过在配置文件中指定docker仓库地址，便于内网快速部署
生成配置文件：kubeadm config print init-defaults ClusterConfiguration > kubeadm.conf
修改kubeadm.conf中的如下两项：
* imageRepository
* kubernetesVersion
修改api服务器地址
* localAPIEndpoint
配置子网网络
* networking下的podSubnet
```

## 6. 拉取k8s必备模块镜像

6.1 查看需要哪些镜像需要拉取

```
kubeadm config images list --config kubeadm.conf
```

6.2 拉取镜像

```
kubeadm config images pull --config ./kubeadm.conf
```

6.3 初始化kubernetes环境

```
sudo kubeadm init --config ./kubeadm.conf
```

## 7. 根据启动成功日志配置

```
1. 创建基本配置
mkdir -p $HOME/.kube
sudo cp -i /etc/kubenetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/conf
2. 设置开机自启
sudo systemctl enable kubelet
sudo systemctl start kubelet
3. 验证启动结果
kubectl get nodes 观察到NotReady 还没配网络环境
4. 部署集群内部通信flannel网络
cd $HOME/working
wget https://raw.githubsercontent.com/coreos/flannel/a70459be0084506e4ec919aa1c114638878db11b/Documentation/kube-flannel.yml
5. 把下载下来的配置文件配置网络
vim kube-flannel.yml
编辑Network与上面podSubnet一致
6. 应用当前配置文件
kubectl apply -f kube-flannel.yml
```

