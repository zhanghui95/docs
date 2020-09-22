# Kubernetes

## 1. k8s概念和架构

### 1.1 概述与特征

```
- 概述
* k8s是谷歌2014年开源的容器化集群管理系统
* 可以使用k8s进行容器化应用部署
* 利于应用扩展
* 部署容器化应用更加简洁和高校

- 特征
* 自动装箱
 基于容器对应用运行环境的资源配置要求自动部署应用容器
* 自我修复
 当容器失败时，会对容器进行重启
 当所部署的Node节点有问题时，会对容器进行重新部署和重新调度
* 水平扩展
 通过简单的命令、用户UI界面或者基于CPU等资源使用情况，对应用容器进行规模扩大或剪裁
* 服务发现
 用户不需要额外的服务发现机制，就能够基于k8s自身能力实现服务发现和负载均衡
* 滚动更新
 可以根据应用的变化，对容器运行的应用进行一次性或批量式更新
* 版本回退
 可以根据应用部署情况，对容器运行的应用进行历史版本即时回退
* 密钥和配置管理
 在不需要重新构建镜像的情况下，可以部署和更新密钥和应用配置，类似热部署
* 存储编排
 自动实现存储系统挂载及应用，特别对有状态应用实现数据持久化非常重要，存储系统可以来自于本地目录、网络存储(NFS、Gluster、Ceph等)、公共云存储服务
* 批处理
 提供一次性任务、定时任务满足批量数据处理和分析的场景
```

### 1.2 集群架构组件

```
* Master主控节点和Node工作节点

Master node
 > API server：集群统一入口，以RESTful方式，交给etcd存储
 > Scheduler：节点调度，选择node节点应用部署
 > Controller-Manager：集中的处理管理，一个资源对应一个控制器
 > Etcd：存储，用于保存集群相关的数据
 
Worker node
 > Kubelet：相当于master指派到node的代表，管理节点的生命周期等
 > Kube-Proxy：提供网络代理，负载均衡等操作
```

### 1.3 核心概念

```
* pod
 > 最小部署单元
 > 一组容器的集合
 > 共享网络
 > 生命周期是短暂的

* controller
 > 确保预期的pod副本数量
 > 无状态应用部署
 > 有状态应用部署

* service
 > 定义一组pod的访问规则
```

## 2. k8s搭建与集群

```
平台规划
单master集群
多master集群
```

### 2.1 kubeadm方式

```
1. 准备三台虚拟机 centos7为例 采用NAT连接
2. 环境准备
 # 关闭防火墙
 systemctl stop firewalld # 临时
 systemctl disable firewalld # 永久
 
 # 关闭selinux
 setenforce 0 # 临时
 sed -i 's/enforcing/disabled/' /etc/selinux/config # 永久
 
 # 关闭swap
 swapoff -a # 临时
 sed -ri 's/.*swap.*/#&/' /etc/fstab # 永久
 
 # 根据规划设置主机名
 hostnamectl set-hostname <hostname>
 
 eg：
 hostnamectl set-hostname master
 hostnamectl set-hostname node1
 hostnamectl set-hostname node2
 
 # 在master添加hosts
 cat >> /etc/hosts << EOF
 192.168.11.1 master
 192.168.11.2 node1
 192.168.11.3 node2
 EOF
 
 # 将桥接的IPV4流量传递到iptables的链
 cat > /etc/sysctl.d/k8s.conf << EOF
 net.bridge.bridge-nf-call-ip6tables = 1
 net.bridge.bridge-nf-call-iptables = 1
 EOF
 
 sysctl --system # 生效
 
 # 时间同步
 yum install ntpdate -y
 ntpdate time.windows.com
3. 所有节点安装docker/kubeadm/kubelet
 3.1 安装docker
 wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -0 /etc/yum.repos.d/docker-ce.repo
 
 yum -y install docker-ce-18.06.1.ce-3.el7
 
 systemctl enable docker && systemctl start docker
 
 docker --version
 
 cat > /etc/docker/daemon.json << EOF
 {
 	"registry-mirrors":["https://b9pmyelo.mirror.aliyuncs.com"]
 }
 EOF
 
 systemctl restart docker
 
 # 设置阿里云yum软件源
 cat > /etc/yum.repos.d/kubernetes.repo << EOF
 [kubernetes]
 name=Kubernetes
 baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
 enable=1
 gpgcheck=0
 repo_gpgcheck=0
 gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
 EOF
 
 3.2 安装kubeadm kubelet kubectl
 由于版本更新频繁，这里制定版本号部署
 yum install -y kubelet-1.18.0 kubeadm-1.18.0 kubectl-1.18.0
 
 systemctl enable kubelet

4. 部署kubernetes master
 kubeadm init \
 -- apiserver-advertise-address=192.168.10.1 \
 --image-repository registry.aliyuncs.com/google_containers \
 -- kubernetes-version v1.18.0 \
 --service-cidr=10.96.0.0/12 \
 --pod-network-cidr=10.244.0.0/16
 
 成功后会给出添加节点的命令提示 复制在其他节点执行，token有效期为24小时 过期后可用下面命令
 kubeadm token create --print-join-command
 
 部署CNI网络插件
 wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
 
 默认景象地址无法访问，sed命令修改为docker hub镜像仓库
 kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
 
 kubectl get pods -n kube-system
 
 测试集群，在集群中创建一个pod验证是否正常运行
 kubectl create deployment nginx --image=nginx
 kubectl expose deployment nginx --port=80 --type=NodePort
 kubectl get pod,svc
 访问地址：http://node ip:port
```

### 2.2 二进制方式





## 3. k8s核心概念



## 4. 搭建集群监控平台



## 5. 搭建高可用集群



## 6. 集群部署项目



















