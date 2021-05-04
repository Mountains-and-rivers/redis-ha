# 环境介绍：
```bash
CentOS： 7.6
Docker： 18.06.1-ce
Kubernetes： 1.13.4
Kuberadm： 1.13.4
Kuberlet： 1.13.4
Kuberctl： 1.13.4
```  
# 部署介绍：

&#8195;创建高可用首先先有一个 Master 节点，然后再让其他服务器加入组成三个 Master 节点高可用，然后再将工作节点 Node 加入。下面将描述每个节点要执行的步骤：
```bash
Master01： 二、三、四、五、六、七、八、九、十一
Master02、Master03： 二、三、五、六、四、九
node01、node02、node03： 二、五、六、九
```
# 集群架构：

  ![kubeadm高可用架构图](https://github.com/Lancger/opsfull/blob/master/images/kubeadm-ha.jpg)
 
# 一、kuberadm 简介

### 1、Kuberadm 作用

&#8195;Kubeadm 是一个工具，它提供了 kubeadm init 以及 kubeadm join 这两个命令作为快速创建 kubernetes 集群的最佳实践。

&#8195;kubeadm 通过执行必要的操作来启动和运行一个最小可用的集群。它被故意设计为只关心启动集群，而不是之前的节点准备工作。同样的，诸如安装各种各样值得拥有的插件，例如 Kubernetes Dashboard、监控解决方案以及特定云提供商的插件，这些都不在它负责的范围。

&#8195;相反，我们期望由一个基于 kubeadm 从更高层设计的更加合适的工具来做这些事情；并且，理想情况下，使用 kubeadm 作为所有部署的基础将会使得创建一个符合期望的集群变得容易。

### 2、Kuberadm 功能
```bash
kubeadm init： 启动一个 Kubernetes 主节点
kubeadm join： 启动一个 Kubernetes 工作节点并且将其加入到集群
kubeadm upgrade： 更新一个 Kubernetes 集群到新版本
kubeadm config： 如果使用 v1.7.x 或者更低版本的 kubeadm 初始化集群，您需要对集群做一些配置以便使用 kubeadm upgrade 命令
kubeadm token： 管理 kubeadm join 使用的令牌
kubeadm reset： 还原 kubeadm init 或者 kubeadm join 对主机所做的任何更改
kubeadm version： 打印 kubeadm 版本
kubeadm alpha： 预览一组可用的新功能以便从社区搜集反馈
```
### 3、功能版本

<table border="0">
    <tr>
        <td><strong>Area<strong></td>
        <td><strong>Maturity Level<strong></td>
    </tr>
    <tr>
        <td>Command line UX</td>
        <td>GA</td>
    </tr>
    <tr>
        <td>Implementation</td>
        <td>GA</td>
    </tr>
    <tr>
        <td>Config file API</td>
        <td>beta</td>
    </tr>
    <tr>
        <td>CoreDNS</td>
        <td>GA</td>
    </tr>
    <tr>
        <td>kubeadm alpha subcommands</td>
        <td>alpha</td>
    </tr>
    <tr>
        <td>High availability</td>
        <td>alpha</td>
    </tr>
    <tr>
        <td>DynamicKubeletConfig</td>
        <td>alpha</td>
    </tr>
    <tr>
        <td>Self-hosting</td>
        <td>alpha</td>
    </tr>
</table>
            
# 二、前期准备

### 1、虚拟机分配说明

<table border="0">
    <tr>
        <td><strong>地址<strong></td>
        <td><strong>主机名</td>
        <td><strong>内存&CPU</td>
        <td><strong>角色</td>
    </tr>
    <tr>
        <td>10.19.2.200</td>
        <td>-</td>
        <td>-</td>
        <td>vip</td>
    </tr>
    <tr>
        <td>10.19.2.56</td>
        <td>k8s-master-01</td>
        <td>2C & 2G</td>
        <td>master</td>
    </tr>
    <tr>
        <td>10.19.2.57</td>
        <td>k8s-master-02</td>
        <td>2C & 2G</td>
        <td>master</td>
    </tr>
    <tr>
        <td>10.19.2.58</td>
        <td>k8s-master-03</td>
        <td>2C & 2G</td>
        <td>master</td>
    </tr>
    <tr>
        <td>10.19.2.246</td>
        <td>k8s-node-01</td>
        <td>4C & 8G</td>
        <td>node</td>
    </tr>
    <tr>
        <td>10.19.2.247</td>
        <td>k8s-node-02</td>
        <td>4C & 8G</td>
        <td>node</td>
    </tr>
    <tr>
        <td>10.19.2.248</td>
        <td>k8s-node-03</td>
        <td>4C & 8G</td>
        <td>node</td>
    </tr>
</table>

### 2、各个节点端口占用

- Master 节点

<table border="0">
    <tr>
        <td><strong>规则<strong></td>
        <td><strong>方向</td>
        <td><strong>端口范围</td>
        <td><strong>作用</td>
        <td><strong>使用者</td>
    </tr>
    <tr>
        <td>TCP</td>
        <td>Inbound 入口</td>
        <td>6443*</td>
        <td>Kubernetes API</td>
        <td>server All</td>
    </tr>
    <tr>
        <td>TCP</td>
        <td>Inbound 入口</td>
        <td>2379-2380</td>
        <td>etcd server</td>
        <td>client API kube-apiserver, etcd</td>
    </tr>
    <tr>
        <td>TCP</td>
        <td>Inbound 入口</td>
        <td>10250</td>
        <td>Kubernetes API</td>
        <td>Self, Control plane</td>
    </tr>
    <tr>
        <td>TCP</td>
        <td>Inbound 入口</td>
        <td>10251</td>
        <td>kube-scheduler</td>
        <td>Self</td>
    </tr>
    <tr>
        <td>TCP</td>
        <td>Inbound 入口</td>
        <td>10252</td>
        <td>kube-controller-manager</td>
        <td>Self</td>
    </tr>
</table>

- node 节点

<table border="0">
    <tr>
        <td><strong>规则<strong></td>
        <td><strong>方向</td>
        <td><strong>端口范围</td>
        <td><strong>作用</td>
        <td><strong>使用者</td>
    </tr>
    <tr>
        <td>TCP</td>
        <td>Inbound 入口</td>
        <td>10250</td>
        <td>Kubernetes API</td>
        <td>Self, Control plane</td>
    </tr>
    <tr>
        <td>TCP</td>
        <td>Inbound 入口</td>
        <td>30000-32767</td>
        <td>NodePort Services**</td>
        <td>All</td>
    </tr>
</table>
    
### 3、基础环境设置

&#8195;Kubernetes 需要一定的环境来保证正常运行，如各个节点时间同步，主机名称解析，关闭防火墙等等。

1、主机名称解析

&#8195;分布式系统环境中的多主机通信通常基于主机名称进行，这在 IP 地址存在变化的可能性时为主机提供了固定的访问人口，因此一般需要有专用的 DNS 服务负责解决各节点主机 不过，考虑到此处部署的是测试集群，因此为了降低系复杂度，这里将基于 hosts 的文件进行主机名称解析。

2、修改hosts和免key登录

```bash
#分别进入不同服务器，进入 /etc/hosts 进行编辑

cat > /etc/hosts << \EOF
127.0.0.1     localhost  localhost.localdomain localhost4 localhost4.localdomain4
::1           localhost  localhost.localdomain localhost6 localhost6.localdomain6
10.19.2.200   k8s-vip         master      master.k8s.io
10.19.2.56    k8s-master-01   master01    master01.k8s.io
10.19.2.57    k8s-master-02   master02    master02.k8s.io
10.19.2.58    k8s-master-03   master03    master03.k8s.io
10.19.2.246   k8s-node-01     node01      node01.k8s.io
10.19.2.247   k8s-node-02     node02      node02.k8s.io
10.19.2.248   k8s-node-03     node03      node03.k8s.io
EOF

#root用户免密登录
mkdir -p /root/.ssh/
chmod 700 /root/.ssh/
echo 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC7bRm20od1b3rzW3ZPLB5NZn3jQesvfiz2p0WlfcYJrFHfF5Ap0ubIBUSQpVNLn94u8ABGBLboZL8Pjo+rXQPkIcObJxoKS8gz6ZOxcxJhldudbadabdanKAAKAKKKKKKKKKKKKKKKKKKKKKKK root@k8s-master-01' > /root/.ssh/authorized_keys
chmod 400 /root/.ssh/authorized_keys
```

3、修改hostname

```bash
#分别进入不同的服务器修改 hostname 名称

# 修改 10.19.2.56 服务器
hostnamectl  set-hostname  k8s-master-01

# 修改 10.19.2.57 服务器
hostnamectl  set-hostname  k8s-master-02

# 修改 10.19.2.58 服务器
hostnamectl  set-hostname  k8s-master-03

# 修改 10.19.2.246 服务器
hostnamectl  set-hostname  k8s-node-01

# 修改 10.19.2.247 服务器
hostnamectl  set-hostname  k8s-node-02

# 修改 10.19.2.248 服务器
hostnamectl  set-hostname  k8s-node-03
```

4、主机时间同步

```bash
#将各个服务器的时间同步，并设置开机启动同步时间服务

systemctl start chronyd.service
systemctl enable chronyd.service
```

5、关闭防火墙服务
```bash
systemctl stop firewalld
systemctl disable firewalld
```

6、关闭并禁用SELinux
```bash
# 若当前启用了 SELinux 则需要临时设置其当前状态为 permissive
setenforce 0

# 编辑／etc/sysconfig selinux 文件，以彻底禁用 SELinux
sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config

# 查看selinux状态
getenforce 

如果为permissive，则执行reboot重新启动即可

```

7、禁用 Swap 设备

&#8195;kubeadm 默认会预先检当前主机是否禁用了 Swap 设备，并在未用时强制止部署 过程因此，在主机内存资惊充裕的条件下，需要禁用所有的 Swap 设备

```
# 关闭当前已启用的所有 Swap 设备
swapoff -a && sysctl -w vm.swappiness=0

sed -ri 's/.*swap.*/#&/' /etc/fstab
或
# 编辑 fstab 配置文件，注释掉标识为 Swap 设备的所有行
vi /etc/fstab

UUID=9be41058-76a6-4588-8e3f-5b44604d8de1 /                       xfs     defaults,noatime        0 0
UUID=4489cc8f-1885-4e17-bfe7-8652fd1d3feb /boot                   xfs     defaults,noatime        0 0
#UUID=0f5ae5f1-4872-471f-9f3a-f172a43fc1ff swap                    swap    defaults,noatime        0 0
```

8、设置系统参数

&#8195;设置允许路由转发，不对bridge的数据进行处理

```bash
#创建 /etc/sysctl.d/k8s.conf 文件

cat > /etc/sysctl.d/k8s.conf << \EOF
vm.swappiness = 0
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

#挂载br_netfilter
modprobe br_netfilter

#生效配置文件
sysctl -p /etc/sysctl.d/k8s.conf

#查看是否生成相关文件
ls /proc/sys/net/bridge
```

9、资源配置文件

`/etc/security/limits.conf` 是 Linux 资源使用配置文件，用来限制用户对系统资源的使用

```bash
echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 65536" >> /etc/security/limits.conf
echo "* soft nproc 65536"  >> /etc/security/limits.conf
echo "* hard nproc 65536"  >> /etc/security/limits.conf
echo "* soft memlock unlimited"  >> /etc/security/limits.conf
echo "* hard memlock unlimited"  >> /etc/security/limits.conf
```

10、安装依赖包以及相关工具

```bash
yum install -y epel-release

yum install -y yum-utils device-mapper-persistent-data lvm2 net-tools conntrack-tools wget vim  ntpdate libseccomp libtool-ltdl
```

# 三、安装Keepalived

- keepalived介绍： 是集群管理中保证集群高可用的一个服务软件，其功能类似于heartbeat，用来防止单点故障
- Keepalived作用： 为haproxy提供vip（10.19.2.200）在三个haproxy实例之间提供主备，降低当其中一个haproxy失效的时对服务的影响。

### 1、yum安装Keepalived
```bash
# 安装keepalived
chattr -i /etc/passwd* && chattr -i /etc/group* && chattr -i /etc/shadow* && chattr -i /etc/gshadow*
yum install -y keepalived
```

### 2、配置Keepalived
```bash
cat <<EOF > /etc/keepalived/keepalived.conf
! Configuration File for keepalived

# 主要是配置故障发生时的通知对象以及机器标识。
global_defs {
   # 标识本节点的字条串，通常为 hostname，但不一定非得是 hostname。故障发生时，邮件通知会用到。
   router_id LVS_k8s
}

# 用来做健康检查的，当时检查失败时会将 vrrp_instance 的 priority 减少相应的值。
vrrp_script check_haproxy {
    script "killall -0 haproxy"   #根据进程名称检测进程是否存活
    interval 3
    weight -2
    fall 10
    rise 2
}

# rp_instance用来定义对外提供服务的 VIP 区域及其相关属性。
vrrp_instance VI_1 {
    state MASTER   #当前节点为MASTER，其他两个节点设置为BACKUP
    interface eth0 #改为自己的网卡
    virtual_router_id 51
    priority 250
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 35f18af7190d51c9f7f78f37300a0cbd
    }
    virtual_ipaddress {
        10.19.2.200   #虚拟ip，即VIP
    }
    track_script {
        check_haproxy
    }

}
EOF
```
当前节点的配置中 state 配置为 MASTER，其它两个节点设置为 BACKUP

```bash
配置说明：

    virtual_ipaddress： vip
    track_script： 执行上面定义好的检测的script
    interface： 节点固有IP（非VIP）的网卡，用来发VRRP包。
    virtual_router_id： 取值在0-255之间，用来区分多个instance的VRRP组播
    advert_int： 发VRRP包的时间间隔，即多久进行一次master选举（可以认为是健康查检时间间隔）。
    authentication： 认证区域，认证类型有PASS和HA（IPSEC），推荐使用PASS（密码只识别前8位）。
    state： 可以是MASTER或BACKUP，不过当其他节点keepalived启动时会将priority比较大的节点选举为MASTER，因此该项其实没有实质用途。
    priority： 用来选举master的，要成为master，那么这个选项的值最好高于其他机器50个点，该项取值范围是1-255（在此范围之外会被识别成默认值100）。
    
# 1、注意防火墙需要放开vrrp协议(不然会出现脑裂现象，三台主机都存在VIP的情况)
#-A INPUT -p vrrp -j ACCEPT
-A RH-Firewall-1-INPUT -p vrrp -j ACCEPT
    
#2、注意上面配置script "killall -0 haproxy"   #根据进程名称检测进程是否存活，会在/var/log/messages每隔一秒执行检测的日志记录
# tail -100f /var/log/message

Sep 27 10:54:16 tw19410s1 Keepalived_vrrp[9113]: /usr/bin/killall -0 haproxy exited with status 1
```

### 3、启动Keepalived
```bash
# 设置开机启动
systemctl enable keepalived

# 启动keepalived
systemctl start keepalived

# 查看启动状态
systemctl status keepalived
```
### 4、查看网络状态

kepplived 配置中 state 为 MASTER 的节点启动后，查看网络状态，可以看到虚拟IP已经加入到绑定的网卡中

```bash
[root@k8s-master-01 ~]# ip address show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:be:86:af brd ff:ff:ff:ff:ff:ff
    inet 10.19.2.56/22 brd 10.19.3.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet 10.19.2.200/32 scope global eth0
       valid_lft forever preferred_lft forever

当关掉当前节点的keeplived服务后将进行虚拟IP转移，将会推选state 为 BACKUP 的节点的某一节点为新的MASTER，可以在那台节点上查看网卡，将会查看到虚拟IP
```

# 四、安装haproxy

&#8195;此处的haproxy为apiserver提供反向代理，haproxy将所有请求轮询转发到每个master节点上。相对于仅仅使用keepalived主备模式仅单个master节点承载流量，这种方式更加合理、健壮。

### 1、yum安装haproxy
```bash
chattr -i /etc/passwd* && chattr -i /etc/group* && chattr -i /etc/shadow* && chattr -i /etc/gshadow*

yum install -y haproxy
```

### 2、配置haproxy
```bash
cat > /etc/haproxy/haproxy.cfg << EOF
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2
    
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon 
       
    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats
#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------  
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000
#---------------------------------------------------------------------
# kubernetes apiserver frontend which proxys to the backends
#--------------------------------------------------------------------- 
frontend kubernetes-apiserver
    mode                 tcp
    bind                 *:16443
    option               tcplog
    default_backend      kubernetes-apiserver    
#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------
backend kubernetes-apiserver
    mode        tcp
    balance     roundrobin
    server      master01.k8s.io   10.19.2.56:6443 check
    server      master02.k8s.io   10.19.2.57:6443 check
    server      master03.k8s.io   10.19.2.58:6443 check
#---------------------------------------------------------------------
# collection haproxy statistics message
#---------------------------------------------------------------------
listen stats
    bind                 *:1080
    stats auth           admin:awesomePassword
    stats refresh        5s
    stats realm          HAProxy\ Statistics
    stats uri            /admin?stats
EOF
```
haproxy配置在其他master节点上(10.19.2.57和10.19.2.58)相同

### 3、启动并检测haproxy
```bash
# 设置开机启动
systemctl enable haproxy

# 开启haproxy
systemctl start haproxy

# 查看启动状态
systemctl status haproxy
```

### 4、检测haproxy端口
```bash
ss -lnt | grep -E "16443|1080"
```

# 五、安装Docker (所有节点)

### 1、移除之前安装过的Docker
```bash
sudo yum remove -y docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-ce-cli \
                  docker-engine
                  
# 查看还有没有存在的docker组件
rpm -qa|grep docker

# 有则通过命令 yum -y remove XXX 来删除,比如：
yum remove docker-ce-cli
```

### 2、配置docker的yum源

下面两个镜像源选择其一即可，由于官方下载速度比较慢，推荐用阿里镜像源

- 阿里镜像源

```bash
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

- Docker官方镜像源
```bash
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

### 3、安装Docker：

```
# 显示docker-ce所有可安装版本：
yum list docker-ce --showduplicates | sort -r

# 安装指定docker版本
sudo yum install docker-ce-18.06.1.ce-3.el7 -y

# 启动docker并设置docker开机启动
systemctl enable docker
systemctl start docker

# 确认一下iptables
确认一下iptables filter表中FOWARD链的默认策略(pllicy)为ACCEPT。

iptables -nvL

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DOCKER-USER  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
    0     0 DOCKER-ISOLATION-STAGE-1  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
    0     0 ACCEPT     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
    0     0 DOCKER     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0           
    0     0 ACCEPT     all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0           
    0     0 ACCEPT     all  --  docker0 docker0  0.0.0.0/0            0.0.0.0/0      
    
Docker从1.13版本开始调整了默认的防火墙规则，禁用了iptables filter表中FOWARD链，这样会引起Kubernetes集群中跨Node的Pod无法通信。但这里通过安装docker 1806，发现默认策略又改回了ACCEPT，这个不知道是从哪个版本改回的，因为我们线上版本使用的1706还是需要手动调整这个策略的。

# 执行下面命令
iptables -P FORWARD ACCEPT

# 修改docker的配置
vim /usr/lib/systemd/system/docker.service

# 增加下面命令
ExecStartPost=/usr/sbin/iptables -P FORWARD ACCEPT

# 配置docker加速器
cat > /etc/docker/daemon.json << \EOF
{
  "registry-mirrors": [
    "https://dockerhub.azk8s.cn",
    "https://i37dz0y4.mirror.aliyuncs.com"
  ],
  "insecure-registries": ["reg.hub.com"]
}
EOF

# 重启Docker
systemctl daemon-reload
systemctl restart docker
```
### 4、docker最终的服务文件
```
#注意，有变量的地方需要使用转义符号

cat > /usr/lib/systemd/system/docker.service << EOF
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
BindsTo=containerd.service
After=network-online.target firewalld.service containerd.service
Wants=network-online.target
Requires=docker.socket

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --exec-opt native.cgroupdriver=systemd
ExecReload=/bin/kill -s HUP \$MAINPID
ExecStartPost=/usr/sbin/iptables -P FORWARD ACCEPT
TimeoutSec=0
RestartSec=2
Restart=always

# Note that StartLimit* options were moved from "Service" to "Unit" in systemd 229.
# Both the old, and new location are accepted by systemd 229 and up, so using the old location
# to make them work for either version of systemd.
StartLimitBurst=3

# Note that StartLimitInterval was renamed to StartLimitIntervalSec in systemd 230.
# Both the old, and new name are accepted by systemd 230 and up, so using the old name to make
# this option work for either version of systemd.
StartLimitInterval=60s

# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity

# Comment TasksMax if your systemd version does not support it.
# Only systemd 226 and above support this option.
TasksMax=infinity

# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes

# kill only the docker process, not all processes in the cgroup
KillMode=process

[Install]
WantedBy=multi-user.target
EOF
```

# 六、安装kubeadm、kubelet

### 1、配置可用的国内yum源用于安装：
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

### 2、安装kubelet

```
# 需要在每台机器上都安装以下的软件包：
     kubeadm: 用来初始化集群的指令。
     kubelet: 在集群中的每个节点上用来启动 pod 和 container 等。
     kubectl: 用来与集群通信的命令行工具。

# 查看kubelet版本列表
yum list kubelet --showduplicates | sort -r 

# 安装kubelet
yum install -y kubelet-1.13.4-0

# 启动kubelet并设置开机启动
systemctl enable kubelet 
systemctl start kubelet

# 检查状态
检查状态,发现是failed状态，正常，kubelet会10秒重启一次，等初始化master节点后即可正常
systemctl status kubelet
```

### 3、安装kubeadm

```
# 负责初始化集群
# 1、查看kubeadm版本列表
yum list kubeadm --showduplicates | sort -r 

# 2、安装kubeadm
yum install -y kubeadm-1.13.4-0
# 安装 kubeadm 时候会默认安装 kubectl ，所以不需要单独安装kubectl

# 3、重启服务器
为了防止发生某些未知错误，这里我们重启下服务器，方便进行后续操作
reboot
```

# 七、初始化第一个kubernetes master节点

```
# 因为需要绑定虚拟IP，所以需要首先先查看虚拟IP启动这几台master机子哪台上

[root@k8s-master-01 ~]# ip address show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:be:86:af brd ff:ff:ff:ff:ff:ff
    inet 10.19.2.56/22 brd 10.19.3.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet 10.19.2.200/32 scope global eth0
       valid_lft forever preferred_lft forever

可以看到虚拟IP 10.19.2.200  和 服务器IP 10.19.2.56在一台机子上，所以初始化kubernetes第一个master要在master01机子上进行安装
```

### 1、创建kubeadm配置的yaml文件
```
# 1、创建kubeadm配置的yaml文件

cat > kubeadm-config.yaml << EOF
apiServer:
  certSANs:
    - k8s-master-01
    - k8s-master-02
    - k8s-master-03
    - master.k8s.io
    - 10.19.2.56
    - 10.19.2.57
    - 10.19.2.58
    - 10.19.2.200
    - 127.0.0.1
  extraArgs:
    authorization-mode: Node,RBAC
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta1
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: "master.k8s.io:16443"
controllerManager: {}
dns: 
  type: CoreDNS
etcd:
  local:    
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.13.4
networking: 
  dnsDomain: cluster.local  
  podSubnet: 10.20.0.0/16
  serviceSubnet: 10.10.0.0/16
scheduler: {}
EOF

以下两个地方设置： 
- certSANs： 虚拟ip地址（为了安全起见，把所有集群地址都加上） 
- controlPlaneEndpoint： 虚拟IP:监控端口号

配置说明：

    imageRepository： registry.aliyuncs.com/google_containers (使用阿里云镜像仓库)
    podSubnet： 10.20.0.0/16 (#pod地址池)
    serviceSubnet： 10.10.0.0/16 (#service地址池)
```

### 2、初始化第一个master节点
```
kubeadm init --config kubeadm-config.yaml 
```
日志
```
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join master.k8s.io:16443 --token i77yg1.1eype0c53jsanoge --discovery-token-ca-cert-hash sha256:8f0a817012ab333a057b6a7410e65971be20b95c1b75fc4015f8f3b6785f626f
```
在此处看日志可以知道，通过
```
kubeadm join master.k8s.io:16443 --token i77yg1.1eype0c53jsanoge --discovery-token-ca-cert-hash sha256:8f0a817012ab333a057b6a7410e65971be20b95c1b75fc4015f8f3b6785f626f
```
来让节点加入集群

### 3、配置kubectl环境变量
```bash
# 配置环境变量

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 指令补全

yum install bash-completion -y
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

### 4、查看组件状态
```bash
kubectl get cs

NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok                   
scheduler            Healthy   ok                   
etcd-0               Healthy   {"health": "true"}   

# 查看pod状态
[root@k8s-master-01 ~]# kubectl get pods --namespace=kube-system
NAME                                    READY   STATUS    RESTARTS   AGE
coredns-78d4cf999f-5zt5z                0/1     Pending   0          7m32s    ---coredns没有启动
coredns-78d4cf999f-mkgsx                0/1     Pending   0          7m32s    ---coredns没有启动
etcd-k8s-master-01                      1/1     Running   0          6m39s
kube-apiserver-k8s-master-01            1/1     Running   0          6m43s
kube-controller-manager-k8s-master-01   1/1     Running   0          6m32s
kube-proxy-88s74                        1/1     Running   0          7m32s
kube-scheduler-k8s-master-01            1/1     Running   0          6m45s

可以看到coredns没有启动，这是由于还没有配置网络插件，接下来配置下后再重新查看启动状态
```
# 八、安装网络插件

### 1、配置flannel插件的yaml文件
```bash
cat > kube-flannel.yaml << EOF
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: flannel
rules:
  - apiGroups:
      - ""
    resources:
      - pods
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes/status
    verbs:
      - patch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-system
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.20.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
---
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: kube-flannel-ds-amd64
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      hostNetwork: true
      nodeSelector:
        beta.kubernetes.io/arch: amd64
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image: registry.cn-shenzhen.aliyuncs.com/cp_m/flannel:v0.10.0-amd64
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: registry.cn-shenzhen.aliyuncs.com/cp_m/flannel:v0.10.0-amd64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: true
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
        - name: run
          hostPath:
            path: /run
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: flannel-cfg
          configMap:
            name: kube-flannel-cfg
EOF

“Network”: “10.20.0.0/16”要和kubeadm-config.yaml配置文件中podSubnet: 10.20.0.0/16相同
```

### 2、创建flanner相关role和pod

```
# 应用生效
[root@k8s-master-01 ~]# kubectl apply -f kube-flannel.yaml
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.extensions/kube-flannel-ds-amd64 created

# 等待一会时间，再次查看各个pods的状态
[root@k8s-master-01 ~]# kubectl get pods --namespace=kube-system
NAME                                    READY   STATUS    RESTARTS   AGE
coredns-78d4cf999f-5zt5z                1/1     Running   0          12m    ---coredns启动成功
coredns-78d4cf999f-mkgsx                1/1     Running   0          12m    ---coredns启动成功
etcd-k8s-master-01                      1/1     Running   0          11m
kube-apiserver-k8s-master-01            1/1     Running   0          12m
kube-controller-manager-k8s-master-01   1/1     Running   0          11m
kube-flannel-ds-amd64-7lj6m             1/1     Running   0          13s
kube-proxy-88s74                        1/1     Running   0          12m
kube-scheduler-k8s-master-01            1/1     Running   0          12m
```


# 九、加入集群

### 1、Master加入集群构成高可用
```
复制秘钥到各个节点

在master01 服务器上执行下面命令，将kubernetes相关文件复制到 master02、master03

如果其他节点为初始化第一个master节点，则将该节点的配置文件复制到其余两个主节点，例如master03为第一个master节点，则将它的k8s配置复制到master02和master01。
```
- 复制文件到 master02
```
ssh root@master02.k8s.io mkdir -p /etc/kubernetes/pki/etcd
scp /etc/kubernetes/admin.conf root@master02.k8s.io:/etc/kubernetes
scp /etc/kubernetes/pki/{ca.*,sa.*,front-proxy-ca.*} root@master02.k8s.io:/etc/kubernetes/pki
scp /etc/kubernetes/pki/etcd/ca.* root@master02.k8s.io:/etc/kubernetes/pki/etcd
```
- 复制文件到 master03

```
ssh root@master03.k8s.io mkdir -p /etc/kubernetes/pki/etcd
scp /etc/kubernetes/admin.conf root@master03.k8s.io:/etc/kubernetes
scp /etc/kubernetes/pki/{ca.*,sa.*,front-proxy-ca.*} root@master03.k8s.io:/etc/kubernetes/pki
scp /etc/kubernetes/pki/etcd/ca.* root@master03.k8s.io:/etc/kubernetes/pki/etcd
```
- master节点加入集群

&#8195;master02 和 master03 服务器上都执行加入集群操作

```bash
kubeadm join master.k8s.io:16443 --token i77yg1.1eype0c53jsanoge --discovery-token-ca-cert-hash sha256:8f0a817012ab333a057b6a7410e65971be20b95c1b75fc4015f8f3b6785f626f --experimental-control-plane
```
&#8195;如果加入失败想重新尝试，请输入 kubeadm reset 命令清除之前的设置，重新执行从“复制秘钥”和“加入集群”这两步

&#8195;如果是master加入，请在最后面加上 –experimental-control-plane 这个参数

```bash
# 显示安装过程:

This node has joined the cluster and a new control plane instance was created:

* Certificate signing request was sent to apiserver and approval was received.
* The Kubelet was informed of the new secure connection details.
* Master label and taint were applied to the new node.
* The Kubernetes control plane instances scaled up.
* A new etcd member was added to the local/stacked etcd cluster.

To start administering your cluster from this node, you need to run the following as a regular user:

        mkdir -p $HOME/.kube
        sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config

Run 'kubectl get nodes' to see this node join the cluster.
```
- 配置kubectl环境变量
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 指令补全

yum install bash-completion -y
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

### 2、node节点加入集群

&#8195;除了让master节点加入集群组成高可用外，slave节点也要加入集群中。

&#8195;这里将k8s-node-01、k8s-node-02、k8s-node-03加入集群，进行工作

&#8195;输入初始化k8s master时候提示的加入命令，如下：

```
kubeadm join master.k8s.io:16443 --token i77yg1.1eype0c53jsanoge --discovery-token-ca-cert-hash sha256:8f0a817012ab333a057b6a7410e65971be20b95c1b75fc4015f8f3b6785f626f
```
&#8195;node节点加入，不需要加上 –experimental-control-plane 这个参数

### 3、如果忘记加入集群的token和sha256 (如正常则跳过)

- 显示获取token列表

```
kubeadm token list
```

默认情况下 Token 过期是时间是24小时，如果 Token 过期以后，可以输入以下命令，生成新的 Token

```
kubeadm token create
```

- 获取ca证书sha256编码hash值

```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```

拼接命令
```
kubeadm join master.k8s.io:16443 --token 882ik4.9ib2kb0eftvuhb58 --discovery-token-ca-cert-hash sha256:0b1a836894d930c8558b350feeac8210c85c9d35b6d91fde202b870f3244016a

如果是master加入，请在最后面加上 –experimental-control-plane 这个参数
```

### 4、查看各个节点加入集群情况
```
kubectl get nodes -o wide

```

# 十、从集群中删除 Node

- Master节点：

```
kubectl drain <node name> --delete-local-data --force --ignore-daemonsets
kubectl delete node <node name>
```

- Slave节点：

```
kubeadm reset
```



## 初始化失败
```bash
kubeadm reset
ifconfig cni0 down
ip link delete cni0
ifconfig flannel.1 down
ip link delete flannel.1
rm -rf /var/lib/cni/
rm -rf /var/lib/etcd/*
```

参考资料：

http://www.mydlq.club/article/4/
