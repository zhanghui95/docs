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

```
1. 准备虚拟机
2. 初始化配置(同kubeadm配置)
3. 部署Etcd集群
4. 安装docker
5. 部署master node
6. 部署worker node

详细过程见pdf
```

## 3. k8s核心概念

### 3.1 yaml说明

```yaml
---
# 控制器定义部分
apiVersion: apps/v1
kind: Deployment
metadata:
  name: __PROJECT__
  labels:
    app: __PROJECT__
spec:
  replicas: 1
  selector:
    matchLabels:
      app: __PROJECT__
      tier: visitor_micoservice
  # 被控制对象部分
  template:
    metadata:
      labels:
        app: __PROJECT__
        tier: visitor_micoservice
    spec:
      nodeSelector:
        env: dev
      containers:
      - name: __PROJECT__
        image: openjdk:8-jre
        command:
          - sh
          - '-c'
        args:
          - >-
            cd /work && java 
            -jar __ARTIFACT_ID__.jar 
            --spring.profiles.active=k8s
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: UMASK
          value: "0022"
        - name: TZ
          value: "Asia/Shanghai"
        volumeMounts:
        - name: work
          mountPath: /work
      volumes:
      - name: work
        persistentVolumeClaim:
          claimName: __PROJECT__-work
```

```
apiVersion：API版本
kind：资源类型
metadata：资源元数据
spec：资源规格
replicas：副本数量
selector：标签选择器
template：pod模板
metadata：pod元数据
spec：pod规格
containers：容器配置

如何快速编写一个yml文件
1. 使用kubectl creat命令生成yaml文件
 kubectl create deployment web --image=nginx -o yaml --dry-run
 kubectl create deployment web --image=nginx -o yaml --dty-run > my.yaml
 
2. 从部署后使用kubectl get命令导出
 kubectl get deploy nginx -o=yaml --export > my.yaml
```

### 3.2 pod

```
概念
1. 最小的部署单元
2. pod是一个或多个容器的集合
3. 一个pod中的容器共享网络命名空间
4. pod是短暂的

pod存在的意义
1. 创建容器使用docker 一个docker对应一个容器，一个容器有进程 一个容器运行一个应用程序
2. pod是多进程设计 运行多个应用程序
3. pod存在为了亲密性应用
 > 两个应用之间进行交互
 > 网络之间调用
 > 两个应用需要频繁调用
 
pod实现机制
1. 共享网络
 > 通过pause容器把其他容器加入命名空间(可以理解为根容器，有ip mac port把其他业务容器加入进来 共享这一个网络)
2. 共享存储
 > 通过Volumn数据卷
 
镜像拉取策略
yml中 spec>containers>imagePullPolicy
默认值IfNotPresent 镜像在宿主机上不存在时拉取
Always每次创建Pod都重新拉取
Never永远不主动拉取

pod资源限制
比如运行的pod对核心数 内存有限制
spec>containers>resources>limits：最大限制
spec>containers>resources>requests：调度大小

pod重启机制
spec>restartPolicy
默认值Always：当容器中指退出后，总是重启
OnFailure：当容器异常退出后 退出状态码非0时，才重启容器
Never：当容器终止退出，从不重启

健康检查
livenessProbe存活检查：如果检查失败，将杀死容器，根据pod的重启策略操作
readlinessProbe就绪检查：如果检查失败，会把pod从service endpoints中剔除

创建pod流程
创建pod->进入API Server->存储到etcd
scheduler通过监听api server查看未绑定的pod 尝试为pod分配主机->将绑定结果存储到etcd
kubelet监听apiserver获取自身node上所要运行的pod清单，与自身缓存比较 新增pod
启动pod启动容器->把本节点容器信息 pod信息同步到etcd
 
影响pod调度的属性
1. 上面配置pod资源限制
2. 节点选择器标签
 spec>nodeSelector>env
 // 命令打标签
 kubectl label node node1 env=dev
3. 节点亲和性
 spec>affinity>nodeAffinity
 硬亲和性：必须满足 满足后调度
 软亲和性：尝试满足
4. 污点和污点容忍
 Taint污点：节点不做普通分配调度，是节点属性
 // 查看污点属性
 kubectl describe node node1 | grep Taint
 // 值
 NoScheduler：一定不被调度
 PreferNoScheduler：尽量不被调度
 NotExecute：不会调度，并且会驱逐
 // 添加污点
 kubectl taint node [node] key=value:上面三个值
 // 删除污点
 kubectl taint node [node] key:添加的值-
 
 污点容忍：即使选择NoScheduler也可能会被调度 在yml配置
```

### 3.3 controller

### 3.4 label

### 3.5 volume

### 3.6 pvc、pv

### 3.7 secret

### 3.8 configMap

### 3.9 namespace

### 3.10 service

### 3.11 探针

### 3.12 调度器

### 3.13 集群安全机制RBAC

### 3.14 helm

## 4. 搭建集群监控平台



## 5. 搭建高可用集群



## 6. 集群部署项目



















