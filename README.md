### 
 
环境规划
 
![](https://picx.zhimg.com/80/v2-9ff5cd9ec0dad7214dd38c0815af778b_720w.jpg?source=d16d100b)
 
添加图片注释，不超过 140 字（可选）
 
简单介绍
 
```
Pod网段：      10.0.0.0/16
Service网段：      10.255.0.0/16
操作系统：       Centos7.6
配置：             4h8g
 
> 由master1、master2通过nginx给apiserver服务做负载均衡
> 由master1负责签发证书
> 本文将介绍由二进制部署k8s v1.22.1
> k8s升级至v1.22.5
> k8s添加node节点
 
centos      7.6
kernel      3.10.0-957.el7.x86_64
docker      20.10.7
cfssl       1.6.0
etcd        3.4.13
k8s         1.22.1
pause       3.3
kube-controllers    v3.21.2
calico      v3.21.2 
coredns     1.7.0
```
 
*   所有机器配置
    
 
### 
 
1、配置主机名
 
*   master节点配置
    
 
```
hostnamectl set-hostname master1
hostnamectl set-hostname master2
hostnamectl set-hostname master3
```
 
*   node节点配置
    
 
```
hostnamectl set-hostname node1COPY
```
 
### 
 
2、配置hosts
 
*   所有机器配置
    
 
```
cat >> /etc/hosts << EOF
192.168.1.160 k8s-master1 master1
192.168.1.161 k8s-master2 master2
192.168.1.162 k8s-master3 master3
192.168.1.164 k8s-node1 node1
EOF
```
 
### 
 
3、配置无密码登录
 
*   所有机器配置
    
 
```
ssh-keygen -t rsa 
ssh-copy-id -i .ssh/id_rsa.pub master1
ssh-copy-id -i .ssh/id_rsa.pub master2
ssh-copy-id -i .ssh/id_rsa.pub master3
ssh-copy-id -i .ssh/id_rsa.pub node1
```
 
### 
 
4、环境初始化
 
*   所有机器配置
    
 
```
#添加镜像源
mv -f /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
yum clean all
yum makecache fast
 
#安装软件包
 yum install -y wget ntpdate telnet dstat tmux vim sysstat net-tools perf atop rsync
yum install -y yum-utils device-mapper-persistent-data lrzsz gcc gcc-c++ make cmake libxml2-devel openssl-devel curl curl-devel unzip sudo ntp libaio-devel wget vim ncurses-devel autoconf automake zlib-devel  python-devel epel-release openssh-server socat  ipvsadm conntrack  rsync
 
#禁用selinux和防火墙
sed -i.bak 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
setenforce 0
systemctl disable --now firewalld
 
#关闭swap分区
    swapoff -a
    sed -ri 's/.*centos-swap/#&/g' /etc/fstab
 
#修改内核参数
modprobe br_netfilter
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sysctl -p /etc/sysctl.d/k8s.conf
 
#安装docker
yum remove docker-ce docker-ce-cli  -y
curl https://releases.rancher.com/install-docker/20.10.sh | sh
systemctl enable --now docker
tee /etc/docker/daemon.json << 'EOF'
{
 "registry-mirrors":\["https://rsbud4vc.mirror.aliyuncs.com","https://registry.docker-cn.com","https://docker.mirrors.ustc.edu.cn","https://dockerhub.azk8s.cn","http://hub-mirror.c.163.com","http://qtid6917.mirror.aliyuncs.com", "https://rncxm540.mirror.aliyuncs.com"\],
  "exec-opts": \["native.cgroupdriver=systemd"\]
}
EOF
systemctl restart docker
 
#时间同步
ln -snf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
/usr/sbin/ntpdate pool.ntp.org
cat >>  /var/spool/cron/root  << EOF
#time sync by bks
*/10 * * * * /usr/sbin/ntpdate pool.ntp.org && /sbin/hwclock -w >/dev/null 2>&1
EOF
 
#安装iptables
    yum install iptables-services -y
    systemctl disable --now iptables
    iptables -F
 
#开启ipvs
cat  >> /etc/sysconfig/modules/ipvs.modules << 'EOF'
#!/bin/bash
ipvs\_modules="ip\_vs ip\_vs\_lc ip\_vs\_wlc ip\_vs\_rr ip\_vs\_wrr ip\_vs\_lblc ip\_vs\_lblcr ip\_vs\_dh ip\_vs\_sh ip\_vs\_nq ip\_vs\_sed ip\_vs\_ftp nf_conntrack"
for kernel\_module in ${ipvs\_modules}; do
 /sbin/modinfo -F filename ${kernel_module} > /dev/null 2>&1
 if \[ 0 -eq 0 \]; then
 /sbin/modprobe ${kernel_module}
 fi
done
EOF
chmod 755 /etc/sysconfig/modules/ipvs.modules
bash /etc/sysconfig/modules/ipvs.modules
```
 
### 
 
5、安装自签证书
 
*   master1机器操作
    
 
### 
 
1、安装cfssl
 
```
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.0/cfssl\_1.6.0\_linux_amd64 /usr/bin/cfssl
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.0/cfssljson\_1.6.0\_linux_amd64   /usr/bin/cfssljson
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.0/cfssl-certinfo\_1.6.0\_linux_amd64  /usr/bin/cfssl-certinfo
chmod +x /usr/bin/cfssl*
```
 
### 
 
2、配置ca证书
 
*   生成ca证书请求文件
    
 
```
\[root@master1 ~\]# mkdir  /opt/certs
\[root@master1 ~\]# cd /opt/certs/
 
#\[root@master1 certs\]# cat > ca-csr.json    << 'EOF'
{
  "CN": "kubernetes",
  "key": {
      "algo": "rsa",
      "size": 2048
  },
 "names": \[
    {
      "C": "CN",
      "ST": "Guangdong",
      "L": "Shenzhen",
      "O": "k8s",
      "OU": "system"
    }
  \],
  "ca": {
          "expiry": "175200h"
  }
}
EOF
```
 
CN:Common Name，浏览器使用该字段验证网站是否合法，一般写的是域名。非常重要。 C：Country。国家 ST：State，州，省 L：Locality，城区，城市 O：Organization Name，组织名称，公司名称 OU：Organization Unit Name。组织单位名称，公司部门 expiry：过期时间。这里填写的是20年
 
*   生成ca证书和私钥
    
 
```
\[root@master1 certs\]#  cfssl gencert -initca ca-csr.json  | cfssljson -bare ca
2022/01/04 17:41:11 \[INFO\] generating a new CA key and certificate from CSR
2022/01/04 17:41:11 \[INFO\] generate received request
2022/01/04 17:41:11 \[INFO\] received CSR
2022/01/04 17:41:11 \[INFO\] generating key: rsa-2048
2022/01/04 17:41:11 \[INFO\] encoded CSR
2022/01/04 17:41:11 \[INFO\] signed certificate with serial number 9715486093067327692213245248860387486652739267
 
\[root@master1 certs\]# ll
total 16
-rw-r--r-- 1 root root 1005 Jan  4 17:41 ca.csr
-rw-r--r-- 1 root root  259 Jan  4 17:40 ca-csr.json
-rw------- 1 root root 1679 Jan  4 17:41 ca-key.pem
-rw-r--r-- 1 root root 1367 Jan  4 17:41 ca.pem
```
 
二、搭建etcd集群
 
 
--------------
 
```
etcd在kubernetes集群是用来存放数据并通知变动的。Kubernetes中没有用到数据库，它把关键数据都存放在etcd中，这使kubernetes的整体结构变得非常简单。
在kubernetes中，数据是随时发生变化的，比如说用户提交了新任务、增加了新的Node、Node宕机了、容器死掉了等等，都会触发状态数据的变更。状态数据变更之后呢，Master上的kube-scheduler和kube-controller-manager，就会重新安排工作，它们的工作安排结果也是数据。这些变化，都需要及时地通知给每一个组件。
etcd有一个特别好用的特性，可以调用它的api监听其中的数据，一旦数据发生变化了，就会收到通知。
有了这个特性之后，kubernetes中的每个组件只需要监听etcd中数据，就可以知道自己应该做什么。kube-scheduler和kube-controller-manager呢，
也只需要把最新的工作安排写入到etcd中就可以了，不用自己费心去逐个通知了
 
尤其注意生产环境必须使用SSD，而且尽量使用Nvme协议SSD，磁盘io对etcd性能影响极大
```
 
*   master1、master2、master3配置
    
 
### 
 
1、创建基于根证书的config配置文件
 
*   在k8s-ops机器上
    
 
```
\[root@master1 certs\]# pwd
/opt/certs
\[root@master1 certs\]#  cat > ca-config.json << 'EOF'
{
  "signing": {
      "default": {
          "expiry": "175200h"
        },
      "profiles": {
          "kubernetes": {
              "usages": \[
                  "signing",
                  "key encipherment",
                  "server auth",
                  "client auth"
              \],
              "expiry": "175200h"
          }
      }
  }
}
EOF
```
 
### 
 
2、生成etcd证书
 
*   在k8s-ops机器上
    
 
```
#配置etcd证书请求，hosts的ip变成自己etcd所在节点的ip
#注意，这里的hosts需要手动填写ip，不能填写ip段
 
\[root@master1 certs\]# cat > etcd-csr.json << 'EOF'
{
  "CN": "etcd",
  "hosts": \[
    "127.0.0.1",
    "192.168.1.160",
    "192.168.1.161",
    "192.168.1.162",
    "192.168.1.199"
  \],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": \[{
    "C": "CN",
    "ST": "Guangdong",
    "L": "Shenzhen",
    "O": "k8s",
    "OU": "system"
  }\]
}
EOF
 
#上述文件hosts字段中IP为所有etcd节点的集群内部通信IP，可以预留几个，做扩容用。
# -profiles是指定特定的使用场景，比如ca-config.json中的kubernetes区域
 
\[root@master1 certs\]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes etcd-csr.json | cfssljson  -bare etcd
\[root@master1 certs\]# ls etcd*.pem
etcd-key.pem  etcd.pem
```
 
### 
 
3、安装etcd
 
*   在master1机器上操作
    
 
```
\[root@master1 ~\]# mkdir /opt/src
\[root@master1 ~\]# cd /opt/src/
\[root@master1 src\]# wget https://github.com/etcd-io/etcd/releases/download/v3.4.13/etcd-v3.4.13-linux-amd64.tar.gz
\[root@master1 src\]# tar -xf etcd-v3.4.13-linux-amd64.tar.gz  -C /opt
\[root@master1 src\]# mv /opt/etcd-v3.4.13-linux-amd64/ /opt/etcd-v3.4.13
\[root@master1 src\]# ln -s /opt/etcd-v3.4.13 /opt/etcd
\[root@master1 ~\]# ln -s /opt/etcd/etcdctl /usr/local/bin/etcdctl
```
 
### 
 
4、拷贝证书，私钥
 
```
#创建目录
\[root@master1 src\]# mkdir -p /opt/etcd/certs  /data/etcd /data/etcd/etcd-server
 
#创建etcd用户
\[root@master1 certs\]# useradd -s /sbin/nologin  -M etcd
 
#拷贝证书
\[root@master1 certs\]# cp /opt/certs/{ca.pem,etcd-key.pem,etcd.pem} /opt/etcd/certs/
 
#修改权限
\[root@master1 certs\]# chown -R etcd.etcd /opt/etcd/certs /data/etcd /data/etcd/etcd-server
```
 
### 
 
5、部署etcd集群
 
*   创建配置文件
    
 
```
\[root@master1 etcd\]# cat /opt/etcd/etcd.conf
#\[Member\]
ETCD_NAME="etcd1"
ETCD\_DATA\_DIR="/data/etcd/etcd-server"
ETCD\_LISTEN\_PEER_URLS="https://192.168.1.160:2380"
ETCD\_LISTEN\_CLIENT_URLS="https://192.168.1.160:2379,http://127.0.0.1:2379"
#\[Clustering\]
ETCD\_INITIAL\_ADVERTISE\_PEER\_URLS="https://192.168.1.160:2380"
ETCD\_ADVERTISE\_CLIENT_URLS="https://192.168.1.160:2379"
ETCD\_INITIAL\_CLUSTER="etcd1=https://192.168.1.160:2380,etcd2=https://192.168.1.161:2380,etcd3=https://192.168.1.162:2380"
ETCD\_INITIAL\_CLUSTER_TOKEN="etcd-cluster"
ETCD\_INITIAL\_CLUSTER_STATE="new"
 
#注：
ETCD_NAME：节点名称，集群中唯一 
ETCD\_DATA\_DIR：数据目录 
ETCD\_LISTEN\_PEER_URLS：集群通信监听地址 
ETCD\_LISTEN\_CLIENT_URLS：客户端访问监听地址 
ETCD\_INITIAL\_ADVERTISE\_PEER\_URLS：集群通告地址 
ETCD\_ADVERTISE\_CLIENT_URLS：客户端通告地址 
ETCD\_INITIAL\_CLUSTER：集群节点地址
ETCD\_INITIAL\_CLUSTER_TOKEN：集群Token
ETCD\_INITIAL\_CLUSTER_STATE：加入集群的当前状态，new是新集群，existing表示加入已有集群
```
 
*   创建服务启动文件
    
 
```
\[root@master1 etcd\]# cat > /usr/lib/systemd/system/etcd.service << 'EOF'
\[Unit\]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
 
\[Service\]
Type=notify
User=etcd
Group=etcd
EnvironmentFile=-/opt/etcd/etcd.conf
WorkingDirectory=/data/etcd/
ExecStart=/opt/etcd/etcd \
  --cert-file=/opt/etcd/certs/etcd.pem \
  --key-file=/opt/etcd/certs/etcd-key.pem \
  --trusted-ca-file=/opt/etcd/certs/ca.pem \
  --peer-cert-file=/opt/etcd/certs/etcd.pem \
  --peer-key-file=/opt/etcd/certs/etcd-key.pem \
  --peer-trusted-ca-file=/opt/etcd/certs/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth
Restart=on-failure
RestartSec=5
LimitNOFILE=65536
 
\[Install\]
WantedBy=multi-user.target
EOF
```
 
*   上传至master2、master3
    
 
```
\[root@master1 opt\]# cd /opt
\[root@master1 opt\]#  for i in master2 master3;do rsync -aP etcd* $i:/opt;done
\[root@master1 opt\]#  for i in master2 master3;do rsync -aP /usr/lib/systemd/system/etcd.service $i:/usr/lib/systemd/system/;done
```
 
*   启动etcd集群
    
 
```
#创建目录
\[root@master2 ~\]# mkdir -p /data/etcd /data/etcd/etcd-server
\[root@master2 ~\]# useradd -s /sbin/nologin  -M etcd
\[root@master2 ~\]# chown -R etcd.etcd /opt/etcd/certs /data/etcd /data/etcd/etcd-server
 
\[root@master3 ~\]# mkdir -p /data/etcd /data/etcd/etcd-server
\[root@master3 ~\]# useradd -s /sbin/nologin  -M etcd
\[root@master3 ~\]# chown -R etcd.etcd /opt/etcd/certs /data/etcd /data/etcd/etcd-server
 
#修改配置文件
\[root@master2 ~\]# cat /opt/etcd/etcd.conf   
#\[Member\]
ETCD_NAME="etcd2"
ETCD\_DATA\_DIR="/data/etcd/etcd-server"
ETCD\_LISTEN\_PEER_URLS="https://192.168.1.161:2380"
ETCD\_LISTEN\_CLIENT_URLS="https://192.168.1.161:2379,http://127.0.0.1:2379"
#\[Clustering\]
ETCD\_INITIAL\_ADVERTISE\_PEER\_URLS="https://192.168.1.161:2380"
ETCD\_ADVERTISE\_CLIENT_URLS="https://192.168.1.161:2379"
ETCD\_INITIAL\_CLUSTER="etcd1=https://192.168.1.160:2380,etcd2=https://192.168.1.161:2380,etcd3=https://192.168.1.162:2380"
ETCD\_INITIAL\_CLUSTER_TOKEN="etcd-cluster"
ETCD\_INITIAL\_CLUSTER_STATE="new"
 
\[root@master3 ~\]# cat /opt/etcd/etcd.conf
#\[Member\]
ETCD_NAME="etcd3"
ETCD\_DATA\_DIR="/data/etcd/etcd-server"
ETCD\_LISTEN\_PEER_URLS="https://192.168.1.162:2380"
ETCD\_LISTEN\_CLIENT_URLS="https://192.168.1.162:2379,http://127.0.0.1:2379"
#\[Clustering\]
ETCD\_INITIAL\_ADVERTISE\_PEER\_URLS="https://192.168.1.162:2380"
ETCD\_ADVERTISE\_CLIENT_URLS="https://192.168.1.162:2379"
ETCD\_INITIAL\_CLUSTER="etcd1=https://192.168.1.160:2380,etcd2=https://192.168.1.161:2380,etcd3=https://192.168.1.162:2380"
ETCD\_INITIAL\_CLUSTER_TOKEN="etcd-cluster"
ETCD\_INITIAL\_CLUSTER_STATE="new"
 
\[root@master1 \]# systemctl daemon-reload
\[root@master1 \]# systemctl enable --now etcd.service
 
\[root@master2 \]# systemctl daemon-reload
\[root@master2 \]# systemctl enable --now etcd.service
 
启动etcd的时候，先启动master1的etcd服务，会一直卡住在启动的状态，然后接着再启动master2的etcd，这样master1这个节点etcd才会正常起来
 
\[root@master3 \]# systemctl daemon-reload
\[root@master3 \]# systemctl enable --now etcd.service
 
\[root@master1\]# systemctl status etcd
\[root@master2\]# systemctl status etcd
\[root@master3\]# systemctl status etcdCOPY
\[root@master1 etcd\]# cd /opt/etcd
\[root@master1 etcd\]# ETCDCTL_API=3
\[root@master1 etcd\]# ./etcdctl --write-out=table --cacert=./certs/ca.pem --cert=./certs/etcd.pem --key=./certs/etcd-key.pem --endpoints=https://192.168.1.160:2379,https://192.168.1.161:2379,https://192.168.1.162:2379  endpoint health
+----------------------------+--------+-------------+-------+
|          ENDPOINT          | HEALTH |    TOOK     | ERROR |
+----------------------------+--------+-------------+-------+
| https://192.168.1.160:2379 |   true | 20.748424ms |       |
| https://192.168.1.161:2379 |   true | 26.082827ms |       |
| https://192.168.1.162:2379 |   true | 29.510983ms |       |
 
#如下，1.162为主节点
\[root@master1 etcd\]# ./etcdctl --write-out=table --cacert=./certs/ca.pem --cert=./certs/etcd.pem --key=./certs/etcd-key.pem --endpoints=https://192.168.1.160:2379,https://192.168.1.161:2379,https://192.168.1.162:2379  endpoint status
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|          ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://192.168.1.160:2379 | 8985c5debfc1e55c |  3.4.13 |   20 kB |     false |      false |        12 |         15 |                 15 |        |
| https://192.168.1.161:2379 | a7ea2df634f83903 |  3.4.13 |   20 kB |     false |      false |        12 |         15 |                 15 |        |
| https://192.168.1.162:2379 | 36c90395130636a0 |  3.4.13 |   20 kB |      true |      false |        12 |         15 |                 15 |        |
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+-------------
```
 
*   etcd启动后，服务日志如下，不影响服务使用
    
 
```
Jan 10 14:23:22 master1 etcd\[28451\]: read-only range request "key:\\"/registry/services/endpoints/default/kubernetes\\" " with result "range\_response\_count:1 size:442" took too long (108.727721ms) to executeCOPY
```
 
三、安装控制节点组件
 
 
--------------
 
> 控制节点配置
 
### 
 
1、下载安装包
 
```
\[root@master1 etcd\]# cd /opt/src/
\[root@master1 src\]# wget https://dl.k8s.io/v1.22.1/kubernetes-server-linux-amd64.tar.gz
\[root@master1 src\]# tar -xf kubernetes-server-linux-amd64.tar.gz
\[root@master1 src\]# cd kubernetes/server/bin/
\[root@master1 src\]# mkdir /opt/kubernetes-v1.22.1
\[root@master1 src\]# ln -s /opt/kubernetes-v1.22.1 /opt/kubernetes 
\[root@master1 bin\]# cp kube-apiserver kube-controller-manager kube-scheduler kubectl /opt/kubernetes 
\[root@master1 ~\]# ln -s /opt/kubernetes/kubectl /usr/local/bin/kubectl
 
\[root@master1 bin\]# rsync -aP kubelet kube-proxy node1:/opt/kubernetes-v1.22.1
\[root@master1 bin\]# mkdir -p /opt/kubernetes/certs
\[root@master1 bin\]# mkdir -p /data/logs/kubernetes/{kube-apiserver,kube-controller-manager,kube-scheduler}
 
#其他master节点创建对应目录
\[root@master2 \]# mkdir -p /data/logs/kubernetes/{kube-apiserver,kube-controller-manager,kube-scheduler}
\[root@master3 \]# mkdir -p /data/logs/kubernetes/{kube-apiserver,kube-controller-manager,kube-scheduler}
 
#node节点做软链接
\[root@node1 ~\]# ln -s /opt/kubernetes-v1.22.1/ /opt/kubernetes
```
 
### 
 
2、部署apiserver组件
 
### 
 
1、创建csr请求文件
 
*   在机器master1操作
    
 
```
\[root@master1 certs\]# cd /opt/certs/
\[root@master1 certs\]# cat >  kube-apiserver-csr.json << 'EOF'
{
  "CN": "kubernetes",
  "hosts": \[
    "127.0.0.1",
    "192.168.1.160",
    "192.168.1.161",
    "192.168.1.162",
    "192.168.1.199",
    "10.255.0.1",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  \],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": \[
    {
      "C": "CN",
      "ST": "Guangdong",
      "L": "Shenzhen",
      "O": "system:masters",
      "OU": "system"
    }
  \]
}
EOF
 
#注： 如果 hosts 字段不为空则需要指定授权使用该证书的 IP 或域名列表。 由于该证书后续被 kubernetes master 集群使用，需要将master节点的IP都填上，同时还需要填写 service 网络的首个IP。(一般是 kube-apiserver 指定的 service-cluster-ip-range 网段的第一个IP，如 10.255.0.1)
 
Work节点的证书使用API授权，不自己签发，所以这里的IP地址除了Work节点不用写，其他都要写。
```
 
### 
 
2、生成证书
 
*   在机器master1操作
    
 
```
\[root@k8s-ops certs\]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-apiserver-csr.json | cfssljson -bare kube-apiserver
\[root@k8s-ops certs\]# ll -h | grep "kube-apiserver*"
-rw-r--r-- 1 root root 1.3K Jan  5 10:53 kube-apiserver.csr
-rw-r--r-- 1 root root  524 Jan  5 10:52 kube-apiserver-csr.json
-rw------- 1 root root 1.7K Jan  5 10:53 kube-apiserver-key.pem
-rw-r--r-- 1 root root 1.7K Jan  5 10:53 kube-apiserver.pem
```
 
### 
 
3、创建token.csv文件
 
*   master1操作
    
 
```
\[root@master1 bin\]# cd /opt/kubernetes/
\[root@master1 kubernetes\]# cat > token.csv << EOF
$(head -c 16 /dev/urandom | od -An -t x | tr -d ' '),kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF
 
#格式：token，用户名，UID，用户组
```
 
### 
 
4、创建api-server的配置文件，替换成自己的ip
 
*   master1操作
    
 
```
\[root@master1 kubernetes\]#  cat kube-apiserver.conf    
KUBE\_APISERVER\_OPTS="--enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
  --anonymous-auth=false \
  --bind-address=192.168.1.160 \
  --secure-port=6443 \
  --advertise-address=192.168.1.160 \
  --insecure-port=0 \
  --authorization-mode=Node,RBAC \
  --runtime-config=api/all=true \
  --enable-bootstrap-token-auth \
  --service-cluster-ip-range=10.255.0.0/16 \
  --token-auth-file=/opt/kubernetes/token.csv \
  --service-node-port-range=30000-50000 \
  --tls-cert-file=/opt/kubernetes/certs/kube-apiserver.pem  \
  --tls-private-key-file=/opt/kubernetes/certs/kube-apiserver-key.pem \
  --client-ca-file=/opt/kubernetes/certs/ca.pem \
  --kubelet-client-certificate=/opt/kubernetes/certs/kube-apiserver.pem \
  --kubelet-client-key=/opt/kubernetes/certs/kube-apiserver-key.pem \
  --service-account-key-file=/opt/kubernetes/certs/ca-key.pem \
  --service-account-signing-key-file=/opt/kubernetes/certs/ca-key.pem  \
  --service-account-issuer=https://kubernetes.default.svc.cluster.local \
  --etcd-cafile=/opt/etcd/certs/ca.pem \
  --etcd-certfile=/opt/etcd/certs/etcd.pem \
  --etcd-keyfile=/opt/etcd/certs/etcd-key.pem \
  --etcd-servers=https://192.168.1.160:2379,https://192.168.1.161:2379,https://192.168.1.162:2379 \
  --enable-swagger-ui=true \
  --enable-aggregator-routing=true \
  --allow-privileged=true \
  --apiserver-count=3 \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/data/logs/kubernetes/kube-apiserver-audit.log \
  --event-ttl=1h \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/data/logs/kubernetes/kube-apiserver \
  --v=2"COPY
#注： 
--logtostderr：启用日志 
--v：日志等级 
--log-dir：日志目录 
--etcd-servers：etcd集群地址 
--bind-address：监听地址 
--secure-port：https安全端口 
--advertise-address：集群通告地址 
--allow-privileged：启用授权 
--service-cluster-ip-range：Service虚拟IP地址段 
--enable-admission-plugins：准入控制模块 
--authorization-mode：认证授权，启用RBAC授权和节点自管理 
--enable-bootstrap-token-auth：启用TLS bootstrap机制 
--token-auth-file：bootstrap token文件 
--service-node-port-range：Service nodeport类型默认分配端口范围 
--kubelet-client-xxx：apiserver访问kubelet客户端证书 
--tls-xxx-file：apiserver https证书 
--etcd-xxxfile：连接Etcd集群证书 –
-audit-log-xxx：审计日志
```
 
### 
 
5、创建服务启动文件
 
```
\[root@master1 kubernetes\]# cat > /usr/lib/systemd/system/kube-apiserver.service << 'EOF'
\[Unit\]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=etcd.service
Wants=etcd.service
 
\[Service\]
EnvironmentFile=-/opt/kubernetes/kube-apiserver.conf
ExecStart=/opt/kubernetes/kube-apiserver $KUBE\_APISERVER\_OPTS
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536
 
\[Install\]
WantedBy=multi-user.target
EOF
 
\[root@master1 ~\]# cp /opt/certs/ca*.pem /opt/kubernetes/certs/
\[root@master1 ~\]# cp /opt/certs/kube-apiserver*.pem /opt/kubernetes/certs/
\[root@master1 ~\]# rsync -aP /opt/kubernetes* master2:/opt
\[root@master1 ~\]# rsync -aP /opt/kubernetes* master3:/opt
\[root@master1 ~\]# for i in master2 master3;do rsync -aP /usr/lib/systemd/system/kube-apiserver.service $i:/usr/lib/systemd/system/;done
```
 
注：master2和master3配置文件kube-apiserver.conf的IP地址修改为实际的[本机IP](https://link.zhihu.com/?target=https%3A//so.csdn.net/so/search%3Fq%3D%25E6%259C%25AC%25E6%259C%25BAIP%26spm%3D1001.2101.3001.7020)
 
```
\[root@master2 ~\]# cat /opt/kubernetes/kube-apiserver.conf         
KUBE\_APISERVER\_OPTS="--enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
  --anonymous-auth=false \
  --bind-address=192.168.1.161 \
  --secure-port=6443 \
  --advertise-address=192.168.1.161 \
 
\[root@master3 ~\]# cat /opt/kubernetes/kube-apiserver.conf         
KUBE\_APISERVER\_OPTS="--enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
  --anonymous-auth=false \
  --bind-address=192.168.1.162 \
  --secure-port=6443 \
  --advertise-address=192.168.1.162 \
```
 
### 
 
6、启动服务
 
```
systemctl daemon-reload
systemctl enable --now kube-apiserver
#systemctl status kube-apiserver
Active: active (running)
```
 
*   验证
    
 
```
\[root@master1 kubernetes\]# curl --insecure https://192.168.1.160:6443/
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
 
  },
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
```
 
*   日志[查看日志](https://link.zhihu.com/?target=https%3A//so.csdn.net/so/search%3Fq%3D%25E6%259F%25A5%25E7%259C%258B%25E6%2597%25A5%25E5%25BF%2597%26spm%3D1001.2101.3001.7020)出现如下报错，这是正常的，github中有对应issue，还未彻底解决 \[root@master1 ~\]# cat /data/logs/kubernetes/kube-apiserver/kube-apiserver.ERROR Log file created at: 2022/01/10 10:47:11 Running on machine: master1 Binary: Built with gc go1.16.7 for linux/amd64 Log line format: \[IWEF\]mmdd hh:mm:ss.uuuuuu threadid file:line\] msg E0110 10:47:11.700932 28617 controller.go:152\] Unable to remove old endpoints from kubernetes service: StorageError: key not found, Code: 1, Key: /registry/masterleases/192.168.1.160, ResourceVersion: 0, AdditionalErrorMsg: \[root@master1 ~\]#
    
 
### 
 
3、部署kubectl
 
### 
 
1、创建csr文件
 
*   master1机器操作
    
 
```
\[root@master1 ~\]# cd /opt/certs/
\[root@master1 certs\]# cat > admin-csr.json  << 'EOF'
{
  "CN": "admin",
  "hosts": \[\],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": \[
    {
      "C": "CN",
      "ST": "Guangdong",
      "L": "Shenzhen",
      "O": "system:masters",             
      "OU": "system"
    }
  \]
}
EOF
 
#注意： 因为希望使用KUBECTL时有完整的集群操作权限，所以本处应配置为值"system:masters"
```
 
### 
 
2、生成证书
 
```
\[root@master1 certs\]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
\[root@master1 certs\]# ll admin*.pem
-rw------- 1 root root 1679 Jan  5 12:11 admin-key.pem
-rw-r--r-- 1 root root 1407 Jan  5 12:11 admin.pemCOPY
```
 
### 
 
3、配置安全上下文
 
*   机器master1操作
    
*   创建kubeconfig配置文件
    
 
```
#设置集群参数
\[root@master1 kubernetes\]# cd /opt/kubernetes
\[root@master1 kubernetes\]# kubectl config set-cluster kubernetes --certificate-authority=/opt/kubernetes/certs/ca.pem --embed-certs=true --server=https://192.168.1.160:6443 --kubeconfig=kube.config
```
 
*   设置客户端认证参数
    
 
```
\[root@master1 kubernetes\]# cp /opt/certs/admin*.pem /opt/kubernetes/certs/
\[root@master1 kubernetes\]# kubectl config set-credentials admin --client-certificate=/opt/kubernetes/certs/admin.pem --client-key=/opt/kubernetes/certs/admin-key.pem --embed-certs=true --kubeconfig=kube.configCOPY
```
 
*   设置上下文参数
    
 
```
\[root@master1 kubernetes\]# kubectl config set-context kubernetes --cluster=kubernetes --user=admin --kubeconfig=kube.configCOPY
```
 
*   设置当前上下文
    
 
```
\[root@master1 kubernetes\]# kubectl config use-context kubernetes --kubeconfig=kube.config
\[root@master1 kubernetes\]# mkdir -p ~/.kube
\[root@master1 kubernetes\]# cp kube.config ~/.kube/configCOPY
```
 
*   授权kubernetes证书访问kubelet api权限
    
 
```
\[root@master1 kubernetes\]# kubectl create clusterrolebinding kube-apiserver:kubelet-apis --clusterrole=system:kubelet-api-admin --user kubernetesCOPY
```
 
### 
 
4、查看集群状态
 
```
\[root@master1 kubernetes\]# kubectl cluster-info
Kubernetes control plane is running at https://192.168.1.160:6443
 
To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
\[root@master1 kubernetes\]# kubectl get componentstatuses 
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS      MESSAGE                                                                                       ERROR
controller-manager   Unhealthy   Get "http://127.0.0.1:10252/healthz": dial tcp 127.0.0.1:10252: connect: connection refused   
scheduler            Unhealthy   Get "http://127.0.0.1:10251/healthz": dial tcp 127.0.0.1:10251: connect: connection refused   
etcd-2               Healthy     {"health":"true"}                                                                             
etcd-0               Healthy     {"health":"true"}                                                                             
etcd-1               Healthy     {"health":"true"}    COPY
```
 
### 
 
5、配置命令补全
 
```
\[root@master1 ~\]# yum install -y bash-completion
\[root@master1 ~\]# source /usr/share/bash-completion/bash_completion
\[root@master1 ~\]# source <(kubectl completion bash)
\[root@master1 ~\]# echo "source <(kubectl completion bash)" >> ~/.bashrcCOPY
```
 
### 
 
4、部署kube-controller-manager组件
 
> controller-manager 作为 [k8s](https://link.zhihu.com/?target=https%3A//so.csdn.net/so/search%3Fq%3Dk8s%26spm%3D1001.2101.3001.7020) 集群的管理控制中心，负责集群内 Node、Namespace、Service、Token、Replication 等资源对象的管理，使集群内的资源对象维持在预期的工作状态。 每一个 controller 通过 api-server 提供的 restful 接口实时监控集群内每个资源对象的状态，当发生故障，导致资源对象的工作状态发生变化，就进行干预，尝试将资源对象从当前状态恢复为预期的工作状态， 常见的 controller 有 Namespace Controller、Node Controller、Service Controller、ServiceAccount Controller、Token Controller、ResourceQuote Controller、Replication Controller等。
 
### 
 
1、创建csr请求文件
 
*   master1机器操作
    
 
```
\[root@master1 kubernetes\]# cd /opt/certs/
\[root@k8s-ops certs\]# cat > kube-controller-manager-csr.json   << 'EOF'
{
    "CN": "system:kube-controller-manager",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "hosts": \[
      "127.0.0.1",
      "192.168.1.160",
      "192.168.1.161",
      "192.168.1.162",
      "192.168.1.199"
    \],
    "names": \[
      {
        "C": "CN",
        "ST": "Guangdong",
        "L": "Shenzhen",
        "O": "system:kube-controller-manager",
        "OU": "system"
      }
    \]
}
EOFCOPY
```
 
注： hosts 列表包含所有 kube-controller-manager 节点 IP； CN 为 system:kube-controller-manager、 O 为 system:kube-controller-manager， kubernetes 内置的 ClusterRoleBindings system:kube-controller-manager 赋予 kube-controller-manager 工作所需的权限
 
### 
 
2、创建ca证书
 
*   master1机器操作
    
 
```
\[root@k8s-ops certs\]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
\[root@k8s-ops certs\]#  ll -h | grep "kube-controller-manager*"
-rw-r--r-- 1 root root 1.2K Jan  5 14:13 kube-controller-manager.csr
-rw-r--r-- 1 root root  422 Jan  5 14:11 kube-controller-manager-csr.json
-rw------- 1 root root 1.7K Jan  5 14:13 kube-controller-manager-key.pem
-rw-r--r-- 1 root root 1.5K Jan  5 14:13 kube-controller-manager.pemCOPY
```
 
### 
 
3、配置安全上下文
 
> master1操作
 
```
#上传证书文件
\[root@master1 kubernetes\]# cd /opt/kubernetes/certs/
\[root@master1 certs\]# cp /opt/certs/kube-controller-manager*.pem ./
 
1.设置集群参数
\[root@master1 certs\]#  kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://192.168.1.160:6443 --kubeconfig=kube-controller-manager.kubeconfig
 
2.设置客户端认证参数
\[root@master1 certs\]# kubectl config set-credentials system:kube-controller-manager --client-certificate=kube-controller-manager.pem --client-key=kube-controller-manager-key.pem --embed-certs=true --kubeconfig=kube-controller-manager.kubeconfig
 
3.设置上下文参数
\[root@master1 certs\]# kubectl config set-context system:kube-controller-manager --cluster=kubernetes --user=system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig
 
4.设置当前上下文
\[root@master1 certs\]# kubectl config use-context system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfigCOPY
```
 
### 
 
4、创建配置文件kube-controller-manager.conf
 
```
\[root@master1 certs\]# cd ..
\[root@master1 kubernetes\]# mv certs/kube-controller-manager.kubeconfig ./
\[root@master1 kubernetes\]# cat > kube-controller-manager.conf  << 'EOF'
KUBE\_CONTROLLER\_MANAGER_OPTS="--cluster-name=kubernetes \
  --secure-port=10257 \
  --bind-address=127.0.0.1 \
  --service-cluster-ip-range=10.255.0.0/16 \
  --allocate-node-cidrs=true \
  --cluster-cidr=10.0.0.0/16 \
  --leader-elect=true \
  --controllers=*,bootstrapsigner,tokencleaner \
  --kubeconfig=/opt/kubernetes/kube-controller-manager.kubeconfig \
  --tls-cert-file=/opt/kubernetes/certs/kube-controller-manager.pem \
  --tls-private-key-file=/opt/kubernetes/certs/kube-controller-manager-key.pem \
  --cluster-signing-cert-file=/opt/kubernetes/certs/ca.pem \
  --cluster-signing-key-file=/opt/kubernetes/certs/ca-key.pem \
  --cluster-signing-duration=175200h0m0s \
  --use-service-account-credentials=true \
  --root-ca-file=/opt/kubernetes/certs/ca.pem \
  --service-account-private-key-file=/opt/kubernetes/certs/ca-key.pem \
  --logtostderr=false \
  --v=2 \
  --log-dir=/data/logs/kubernetes/kube-controller-manager"
EOFCOPY
```
 
*   配置解析
    
 
```
--service-cluster-ip-range=10.255.0.0/16    #表示service 分配的IP
--cluster-cidr=10.0.0.0/16  #表示pod分配的IP
--leader-elect=true     #启用CONTROOLER-MANAGER主节点选举功能
--controllers=*,bootstrapsigner,tokencleaner    #启用完整的所有的控制器
--use-service-account-credentials   #启用K8S内置的RBAC权限策略COPY
```
 
### 
 
5、创建启动文件
 
```
\[root@master1 kubernetes\]# cat > /usr/lib/systemd/system/kube-controller-manager.service << 'EOF'
\[Unit\]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
\[Service\]
EnvironmentFile=-/opt/kubernetes/kube-controller-manager.conf
ExecStart=/opt/kubernetes/kube-controller-manager $KUBE\_CONTROLLER\_MANAGER_OPTS
Restart=on-failure
RestartSec=5
\[Install\]
WantedBy=multi-user.target
EOFCOPY
```
 
### 
 
6、启动服务
 
```
\[root@master1 kubernetes\]# pwd
/opt/kubernetes
\[root@master1 kubernetes\]# for i in master2 master3;do rsync -a certs/kube-controller-manager*.pem $i:/opt/kubernetes/certs/;done
 
\[root@master1 kubernetes\]# for i in master2 master3;do rsync -a kube-controller-manager.{conf,kubeconfig} $i:/opt/kubernetes/;done
 
\[root@master1 kubernetes\]# for i in master2 master3;do rsync -a /usr/lib/systemd/system/kube-controller-manager.service  $i:/usr/lib/systemd/system/;done
 
\[root@master1 kubernetes\]# systemctl daemon-reload 
\[root@master1 kubernetes\]# systemctl enable --now kube-controller-manager
\[root@master1 kubernetes\]# systemctl status kube-controller-manager
  Active: active (running) since 
 
\[root@master2 kubernetes\]# systemctl daemon-reload 
\[root@master2 kubernetes\]# systemctl enable --now kube-controller-manager
\[root@master2 kubernetes\]# systemctl status kube-controller-manager
  Active: active (running) since 
 
\[root@master3 kubernetes\]# systemctl daemon-reload 
\[root@master3 kubernetes\]# systemctl enable --now kube-controller-manager
\[root@master3 kubernetes\]# systemctl status kube-controller-manager
  Active: active (running) since COPY
```
 
### 
 
5、部署kube-scheduler组件
 
> Scheduler负责节点资源管理，接收来自kube-apiserver创建Pods的任务，收到任务后它会检索出所有符合该Pod要求的Node节点（通过预选策略和优选策略），开始执行Pod调度逻辑。调度成功后将Pod绑定到目标节点上
 
### 
 
1、创建csr请求文件
 
*   master1机器操作
    
 
```
\[root@master1 kubernetes\]# cd /opt/certs/
\[root@master1 certs\]#  cat > kube-scheduler-csr.json <<'EOF'
{
    "CN": "system:kube-scheduler",
    "hosts": \[
      "127.0.0.1",
      "192.168.1.160",
      "192.168.1.161",
      "192.168.1.162",
      "192.168.1.199"
    \],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": \[
      {
        "C": "CN",
        "ST": "Guangdong",
        "L": "Shenzhen",
        "O": "system:kube-scheduler",
        "OU": "system"
      }
    \]
}
EOF
```
 
注： hosts 列表包含所有 kube-scheduler 节点 IP； CN 为 system:kube-scheduler、O 为 system:kube-scheduler，kubernetes 内置的 ClusterRoleBindings system:kube-scheduler 将赋予 kube-scheduler 工作所需的权限。
 
### 
 
2、生成ca证书
 
*   master1机器操作
    
 
```
\[root@master1 certs\]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler
\[root@master1 certs\]# ll -h | grep kube-scheduler
-rw-r--r-- 1 root root 1.1K Jan  5 14:55 kube-scheduler.csr
-rw-r--r-- 1 root root  404 Jan  5 14:55 kube-scheduler-csr.json
-rw------- 1 root root 1.7K Jan  5 14:55 kube-scheduler-key.pem
-rw-r--r-- 1 root root 1.5K Jan  5 14:55 kube-scheduler.pem
\[root@master1 certs\]#
```
 
### 
 
3、配置安全上下文
 
*   master1机器操作
    
 
```
\[root@master1 certs\]# cd /opt/kubernetes/certs/
\[root@master1 certs\]# cp /opt/certs/kube-scheduler*.pem ./
 
1.设置集群参数
\[root@master1 certs\]# kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://192.168.1.160:6443 --kubeconfig=kube-scheduler.kubeconfig
 
2.设置客户端认证参数
\[root@master1 certs\]# kubectl config set-credentials system:kube-scheduler --client-certificate=kube-scheduler.pem --client-key=kube-scheduler-key.pem --embed-certs=true --kubeconfig=kube-scheduler.kubeconfig
 
3.设置上下文参数
\[root@master1 certs\]# kubectl config set-context system:kube-scheduler --cluster=kubernetes --user=system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig
 
4.设置当前上下文
\[root@master1 certs\]# kubectl config use-context system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfigCOPY
```
 
### 
 
4、创建配置文件kube-scheduler.conf
 
```
\[root@master1 certs\]# cd ..
\[root@master1 kubernetes\]# mv certs/kube-scheduler.kubeconfig ./
 
\[root@master1 work\]# cat > kube-scheduler.conf << 'EOF'
KUBE\_SCHEDULER\_OPTS="--address=127.0.0.1 \
--kubeconfig=/opt/kubernetes/kube-scheduler.kubeconfig \
--leader-elect=true \
--logtostderr=false \
--log-dir=/data/logs/kubernetes/kube-scheduler \
--v=2"
EOF
```
 
### 
 
5、创建启动文件
 
```
\[root@master1 kubernetes\]#  cat > /usr/lib/systemd/system/kube-scheduler.service << 'EOF'
\[Unit\]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
 
\[Service\]
EnvironmentFile=-/opt/kubernetes/kube-scheduler.conf
ExecStart=/opt/kubernetes/kube-scheduler $KUBE\_SCHEDULER\_OPTS
Restart=on-failure
RestartSec=5
 
\[Install\]
WantedBy=multi-user.target
EOFCOPY
```
 
### 
 
6、启动服务
 
```
\[root@master1 kubernetes\]# for i in master2 master3;do rsync -a certs/kube-scheduler*.pem $i:/opt/kubernetes/certs/;done
 
\[root@master1 kubernetes\]# for i in master2 master3;do rsync -a kube-scheduler.{conf,kubeconfig} $i:/opt/kubernetes;done
 
\[root@master1 kubernetes\]# for i in master2 master3;do rsync -a /usr/lib/systemd/system/kube-scheduler.service  $i:/usr/lib/systemd/system;done
 
\[root@master1 kubernetes\]# systemctl daemon-reload 
\[root@master1 kubernetes\]# systemctl enable --now kube-scheduler
\[root@master1 kubernetes\]# systemctl status kube-scheduler
  Active: active (running) since 
 
\[root@master2 kubernetes\]# systemctl daemon-reload 
\[root@master2 kubernetes\]# systemctl enable --now kube-scheduler
\[root@master2 kubernetes\]# systemctl status kube-scheduler
  Active: active (running) since 
 
\[root@master3 kubernetes\]# systemctl daemon-reload 
\[root@master3 kubernetes\]# systemctl enable --now kube-scheduler
\[root@master3 kubernetes\]# systemctl status kube-scheduler
  Active: active (running) since COPY
```
 
四、安装工作节点组件
 
 
--------------
 
这里并没有打算在master节点运行pod，所以以下操作在node节点运行即可 若需要在master节点运行pod，也按照以下步骤操作即可，但需要注意配置完成后打上污点，避免master节点上运行太多pod
 
*   工作节点配置
    
 
> 由于国内无法拉取[http://k8s.gcr.io](https://link.zhihu.com/?target=http%3A//k8s.gcr.io)镜像可以采用以下方法
 
```
将k8s.gcr.io替换为
registry.cn-hangzhou.aliyuncs.com/google_containers
或者
registry.aliyuncs.com/google_containers
或者
mirrorgooglecontainers
 
### quay.io 地址替换
将 quay.io 替换为 quay-mirror.qiniu.com
 
### gcr.io 地址替换
将 gcr.io 替换为 registry.aliyuncs.com
 
如拉取pause
官方地址拉取：docker pull k8s.gcr.io/pause:3.3
可更换为：docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.3COPY
```
 
### 
 
1、部署kubelet组件
 
> 运行在每个计算节点上 kubelet 组件通过 api-server 提供的接口监测到 kube-scheduler 产生的 pod 绑定事件，然后从 etcd 获取 pod 清单，下载镜像并启动容器。 同时监视分配给该Node节点的 pods，周期性获取容器状态，再通过api-server通知各个组件。 简单地说, kubelet的主要功能就是定时从某个地方获取节点上pod的期望状态(运行什么容器、运行的副本数量、网络或者存储如何配置等等), 并调用对应的容器平台接口达到这个状态定时汇报当前节点的状态给 apiserver,以供调度的时候使用镜像和容器的清理工作, 保证节点上镜像不会占满磁盘空间,退出的容器不会占用太多资源
 
  
 
操作在master1上操作，然后同步至node1
 
  
 
### 
 
1、创建kubelet-bootstrap.kubeconfig
 
  
 
```
\[root@master1 certs\]# cd /opt/kubernetes/certs
\[root@master1 certs\]# BOOTSTRAP_TOKEN=$(awk -F "," '{print $1}' /opt/kubernetes/token.csv)
 
\[root@master1 certs\]#  kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://192.168.1.160:6443 --kubeconfig=kubelet-bootstrap.kubeconfig
 
\[root@master1 certs\]# kubectl config set-credentials kubelet-bootstrap --token=${BOOTSTRAP_TOKEN} --kubeconfig=kubelet-bootstrap.kubeconfig
 
\[root@master1 certs\]# kubectl config set-context default --cluster=kubernetes --user=kubelet-bootstrap --kubeconfig=kubelet-bootstrap.kubeconfig
 
\[root@master1 certs\]# kubectl config use-context default --kubeconfig=kubelet-bootstrap.kubeconfig
 
#\[root@master1 certs\]# kubectl create clusterrolebinding cluster-system-anonymous --clusterrole=cluster-admin --user=kubelet-bootstrap
 
\[root@master1 certs\]# kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
 
\[root@master1 certs\]# cd ..
\[root@master1 kubernetes\]# mv certs/kubelet-bootstrap.kubeconfig ./COPY
```
 
### 
 
2、创建配置文件kubelet.conf
 
“cgroupDriver”: "systemd"要和docker的驱动一致。 address这里填写的是0.0.0.0，方便把配置同步给其他node
 
```
\[root@node1 kubernetes\]# cat > kubelet.conf  <<'EOF'
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 0.0.0.0
port: 10250
readOnlyPort: 0
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /opt/kubernetes/certs/ca.pem
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
cgroupDriver: systemd
clusterDNS:
- 10.255.0.2
clusterDomain: cluster.local
healthzBindAddress: 127.0.0.1
healthzPort: 10248
rotateCertificates: true
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
maxOpenFiles: 1000000
maxPods: 110
EOFCOPY
```
 
### 
 
3、服务启动
 
```
#上传文件至node节点
\[root@master1 kubernetes\]# rsync -a kubelet-bootstrap.kubeconfig kubelet.conf node1:/opt/kubernetes/
\[root@master1 kubernetes\]# rsync -a certs/ca.pem node1:/opt/kubernetes/certs/
COPY
```
 
### 
 
4、node1配置
 
```
\[root@node1 ~\]# cat > /usr/lib/systemd/system/kubelet.service << 'EOF'
\[Unit\]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service
Requires=docker.service
\[Service\]
ExecStart=/opt/kubernetes/kubelet \
  --bootstrap-kubeconfig=/opt/kubernetes/kubelet-bootstrap.kubeconfig \
  --cert-dir=/opt/kubernetes/certs \
  --kubeconfig=/opt/kubernetes/kubelet.kubeconfig \
  --config=/opt/kubernetes/kubelet.conf \
  --network-plugin=cni \
  --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.3 \
  --logtostderr=false \
  --log-dir=/data/logs/kubernetes/kubelet \
  --v=2
Restart=on-failure
RestartSec=5
 
\[Install\]
WantedBy=multi-user.target
EOF
 
#注：
–hostname-override：显示名称，集群中唯一 
–network-plugin：启用CNI 
–kubeconfig：空路径，会自动生成，后面用于连接apiserver 
–bootstrap-kubeconfig：首次启动向apiserver申请证书
–config：配置参数文件 
–cert-dir：kubelet证书生成目录 
–pod-infra-container-image：管理Pod网络容器的镜像COPY
\[root@node1 ~\]# mkdir -p /data/logs/kubernetes/kubelet
\[root@node1 ~\]# ln -s /opt/kubernetes/kubelet /usr/local/bin/kubelet
\[root@node1 ~\]# systemctl daemon-reload
\[root@node1 ~\]# systemctl enable --now kubelet
```
 
### 
 
5、批准请求访问
 
```
#执行如下命令可以看到一个worker节点发送了一个 CSR 请求：
 
\[root@master1 ~\]# kubectl get csr                        
NAME        AGE     SIGNERNAME                                    REQUESTOR           CONDITION
csr-qkkd8   4m38s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
 
\[root@master1 ~\]# kubectl certificate approve csr-qkkd8 
certificatesigningrequest.certificates.k8s.io/csr-qkkd8 approved
 
\[root@master1 ~\]# kubectl get csr
NAME        AGE     SIGNERNAME                                    REQUESTOR           CONDITION
csr-qkkd8   4m54s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Approved,Issued
 
\[root@master1 ~\]# kubectl get nodes
NAME    STATUS     ROLES    AGE     VERSION
node1   NotReady   <none>   4m53s   v1.22.1
 
#注意：STATUS是NotReady是因为还没有安装网络插件
```
 
### 
 
2、部署kube-proxy组件
 
### 
 
1、创建csr请求
 
```
\[root@master1 ~\]# cd /opt/certs/
\[root@k8s-ops certs\]# cat > kube-proxy-csr.json << 'EOF'
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": \[
    {
      "C": "CN",
      "ST": "Guangdong",
      "L": "Shenzhen",
      "O": "k8s",
      "OU": "system"
    }
  \]
}
EOF
```
 
### 
 
2、生成ca证书
 
```
\[root@master1 certs\]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
 
\[root@master1 certs\]# kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://192.168.1.160:6443 --kubeconfig=kube-proxy.kubeconfig 
 
\[root@master1 certs\]# kubectl config set-credentials kube-proxy --client-certificate=kube-proxy.pem --client-key=kube-proxy-key.pem --embed-certs=true --kubeconfig=kube-proxy.kubeconfig
 
\[root@master1 certs\]#  kubectl config set-context default --cluster=kubernetes --user=kube-proxy --kubeconfig=kube-proxy.kubeconfig
 
\[root@master1 certs\]# kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
COPY
```
 
### 
 
3、创建配置文件kube-proxy.conf
 
```
\[root@master1 certs\]# cat > kube-proxy.conf << 'EOF'
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
clientConnection:
  kubeconfig: /opt/kubernetes/kube-proxy.kubeconfig
clusterCIDR: 10.0.0.0/16
healthzBindAddress: 0.0.0.0:10256
metricsBindAddress: 0.0.0.0:10249
mode: "ipvs"
EOFCOPY
```
 
*   配置解析
    
 
```
bindAddress: 0.0.0.0                  # KUBE-PROXY的监听地址\[官方默认"0.0.0.0"\]
clusterCIDR: "10.0.0.0/16"           # CNI网络插件所使用的CIDR范围\[官方：PodIP的CIDR范围\]；
                                      # 与"kube-controller-manager"的"--cluster-cidr="配置有关
healthzBindAddress: "0.0.0.0:10256"   # KUBE-PROXY健康检查的地址\[官方默认"0.0.0.0:10256"\]
metricsBindAddress: "0.0.0.0:10249"   # KUBE-PROXY获取METRICS指标的地址\[官方默认"127.0.0.1:10249"\]
mode: ipvs                            # 所使用流量转发模式COPY
```
 
### 
 
4、服务启动
 
```
\[root@k8s-ops certs\]# cat > kube-proxy.service << 'EOF'
\[Unit\]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target
 
\[Service\]
WorkingDirectory=/opt/kubernetes
ExecStart=/opt/kubernetes/kube-proxy \
  --config=/opt/kubernetes/kube-proxy.conf \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/data/logs/kubernetes/kube-proxy \
  --v=2
Restart=on-failure
RestartSec=5
LimitNOFILE=65536
 
\[Install\]
WantedBy=multi-user.target
EOF
 
#上传文件至node节点
\[root@k8s-ops certs\]# rsync -a kube-proxy.kubeconfig kube-proxy.conf node1:/opt/kubernetes/
\[root@k8s-ops certs\]# rsync -a kube-proxy.service node1:/usr/lib/systemd/system/
COPY
```
 
### 
 
5、node1配置
 
```
\[root@node1 ~\]# mkdir -p /data/logs/kubernetes/kube-proxy
\[root@node1 ~\]# systemctl daemon-reload
\[root@node1 ~\]# systemctl enable --now kube-proxy
\[root@node1 ~\]# systemctl status kube-proxy
   Active: active (running)COPY
```
 
### 
 
3、部署calico组件
 
```
#创建存放yaml文件的目录
\[root@master1 ~\]# mkdir /opt/k8s-yaml/
\[root@master1 ~\]# cd /opt/k8s-yaml/
 
#下载官方默认的yaml文件
\[root@master1 k8s-yaml\]# curl https://docs.projectcalico.org/manifests/calico.yaml -O
 
#修改yaml文件,也可直接使用我修改后上传的yaml文件
\[root@master1 k8s-yaml\]# vim calico.yaml
- name: CALICO\_IPV4POOL\_CIDR       # PodIP的CIDR范围
  value: "10.0.0.0/16"            # \[修改\]PodCIDR范围\[等于"kube-controller-manager"中的"--cluster-cidr="\]
- name: CALICO\_IPV4POOL\_IPIP       # CALICO IPIP 模式的启用/禁用
  value: "Never"                   # \[修改\]"Never"表示使用BGP；"Always"表示使用IPIP
 
- name: IP\_AUTODETECTION\_METHOD    # \[按需使用\]特殊配置，未在默认的配置文件中存在，需要手动增加，与"CALICO\_IPV4POOL\_CIDR"同级
  value: "interface=eth0"          # CALICO的部署默认会自动选择合适的网卡，但在一些环境中此方法可能会存在问题，本配置项的作用为
                                   # 用于定义部署过程中的网卡选择；"eth0"这实际是一个正则表达式，"eth0"表示严格匹配所使用的网卡
                                   # 提示：如果你的网络环境能正常部署成功，不建议增加此项而提高维护的复杂度
 
#部署
\[root@master1 k8s-yaml\]# kubectl apply -f calico.yaml 
 
#查询状态running表示部署成功
\[root@master1 k8s-yaml\]# kubectl get pods -o wide -A 
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE   IP              NODE    NOMINATED NODE   READINESS GATES
kube-system   calico-kube-controllers-6b9fbfff44-5dmb7   1/1     Running   0          22m   10.0.166.128    node1   <none>           <none>
kube-system   calico-node-cf6jx                          1/1     Running   0          21m   192.168.1.164   node1   <none>           <none>
 
#查询node节点状态转换为Ready
\[root@master1 k8s-yaml\]# kubectl get nodes
NAME    STATUS   ROLES    AGE   VERSION
node1   Ready    <none>   16h   v1.22.1
\[root@master1 k8s-yaml\]# COPY
```
 
### 
 
4、部署coredns
 
### 
 
1、部署
 
```
#下载官方默认yaml文件
\[root@master1 k8s-yaml\]# curl https://raw.githubusercontent.com/coredns/deployment/master/kubernetes/coredns.yaml.sed > coredns.yaml
 
#修改yaml文件，也可直接使用我上传的yaml文件
\[root@master1 k8s-yaml\]# vi coredns.yaml
...
data:
  Corefile: |
    .:53 {
        errors
        health {
          lameduck 5s
        }
        ready
        kubernetes CLUSTER\_DOMAIN REVERSE\_CIDRS {   # \[修改\]"CLUSTER_DOMAIN"需要配置为"cluster.local"，这个值实际与K8S的默认定义有关
          fallthrough in-addr.arpa ip6.arpa         # 涉及到K8S的多个配置，所以不建议在K8S的配置过程中修改相关值；
        }                                           # "REVERSE_CIDRS"需要配置为"in-addr.arpa ip6.arpa"；本处的配置涉及的是DNS的反向解释功能
        prometheus :9153
        forward . UPSTREAMNAMESERVER {              # "UPSTREAMNAMESERVER"需要配置为"/etc/resolv.conf"；本处的配置涉及的是DNS的正向解释功能
          max_concurrent 1000                       # 
        }
        cache 30
        loop
        reload
        loadbalance
    }STUBDOMAINS                                     # \[删除\]如果"STUBDOMAINS"有这个东西则删它；猜测是官方表明可以在本处增加子配置项，
                                                     # 新版本的YAML文件中有这个字段\[若不存在则不需要任何操作\]
\-\-\-
...
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: CLUSTER\_DNS\_IP                          # \[修改\]"CLUSTER\_DNS\_IP"需要配置为"10.255.0.2";本处为定义K8S集群内的DNS服务器的地址；
                                                     # 这个值应该与"kubelet.conf"中定义的"clusterDNS"配置项的值相同；
 
#执行                                               
\[root@master1 k8s-yaml\]# kubectl apply -f coredns.yaml 
 
#检查状态
\[root@master1 k8s-yaml\]# kubectl get pods -o wide -A
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE    IP              NODE    NOMINATED NODE   READINESS GATES
kube-system   calico-kube-controllers-6b9fbfff44-5dmb7   1/1     Running   0          44m    10.0.166.128    node1   <none>           <none>
kube-system   calico-node-cf6jx                          1/1     Running   0          43m    192.168.1.164   node1   <none>           <none>
kube-system   coredns-86f4cdc7bc-82xd9                   1/1     Running   0          4m2s   10.0.166.129    node1   <none>           <none>
\[root@master1 k8s-yaml\]# kubectl get svc -n kube-system 
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.255.0.2   <none>        53/UDP,53/TCP,9153/TCP   4m12sCOPY
```
 
### 
 
2、验证coredns
 
```
\[root@master1 k8s-yaml\]# kubectl run busybox --image busybox:1.28 --restart=Never --rm -it busybox -- sh
If you don't see a command prompt, try pressing enter.
/ # ping baidu.com
PING baidu.com (220.181.38.251): 56 data bytes
64 bytes from 220.181.38.251: seq=0 ttl=52 time=37.184 ms
64 bytes from 220.181.38.251: seq=1 ttl=52 time=36.807 ms
64 bytes from 220.181.38.251: seq=2 ttl=52 time=36.656 ms
^C
--- baidu.com ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 36.656/36.882/37.184 ms
/ # ping baidu.com
PING baidu.com (220.181.38.251): 56 data bytes
64 bytes from 220.181.38.251: seq=0 ttl=52 time=37.713 ms
64 bytes from 220.181.38.251: seq=1 ttl=52 time=37.606 ms
^C
--- baidu.com ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 37.606/37.659/37.713 ms
 
/ # nslookup kubernetes.default.svc.cluster.local
Server:    10.255.0.2
Address 1: 10.255.0.2 kube-dns.kube-system.svc.cluster.local
 
Name:      kubernetes.default.svc.cluster.local
Address 1: 10.255.0.1 kubernetes.default.svc.cluster.local
/ # exitCOPY
```
 
五、部署服务测试集群
 
 
--------------
 
```
\[root@master1 k8s-yaml\]# cat > nginx-test.yaml  << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-test
spec:
  selector:
    matchLabels:
      name: nginx-test
  replicas: 2
  template:
    metadata:
      labels:
        name: nginx-test
    spec:
      containers:
      - name: nginx
        image: nginx:1.20.1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
\-\-\-
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    name: nginx-test
spec:
  selector:
    name: nginx-test
  type: NodePort
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
    nodePort: 30380
EOF
 
#执行
\[root@master1 k8s-yaml\]# kubectl apply -f nginx-test.yaml 
 
#访问
\[root@master1 k8s-yaml\]# curl http://192.168.1.164:30380
<!DOCTYPE html>
<body>
<h1>Welcome to nginx!</h1>
\[root@mastCOPY
```
 
六、安装keepalived+nginx实现k8s apiserver高可用
 
 
------------------------------------------
 
*   master1、master2配置
    
 
> 生产环境建议使用云厂商的负载机器
 
### 
 
1、配置nginx主备
 
*   在master1和master2上做nginx主备安装
    
 
```
\[root@master1 ~\]#  yum install nginx nginx-mod-stream keepalived -y
\[root@master2 ~\]#  yum install nginx nginx-mod-stream  keepalived -yCOPY
```
 
*   修改nginx配置文件。主备一样
    
*   master1、master2均配置
    
 
```
\[root@master1 nginx\]# vi /etc/nginx/nginx.conf #在配置文件最后添加
stream {
    upstream kube-apiserver {
        server 192.168.1.160:6443     max\_fails=3 fail\_timeout=30s;
        server 192.168.1.161:6443     max\_fails=3 fail\_timeout=30s;
        server 192.168.1.162:6443     max\_fails=3 fail\_timeout=30s;
    }
    server {
        listen 16443;
        proxy\_connect\_timeout 2s;
        proxy_timeout 900s;
        proxy_pass kube-apiserver;
    }
}COPY
```
 
### 
 
2、keepalive配置
 
### 
 
1、主keepalived
 
*   master1配置
    
 
```
\[root@master1 ssl\]# cat > /etc/keepalived/keepalived.conf <<'EOF'
global_defs { 
   notification_email { 
     acassen@firewall.loc 
     failover@firewall.loc 
     sysadmin@firewall.loc 
   } 
   notification\_email\_from Alexandre.Cassen@firewall.loc  
   smtp_server 127.0.0.1 
   smtp\_connect\_timeout 30 
   router\_id NGINX\_MASTER
} 
 
vrrp\_script check\_nginx {
    script "/etc/keepalived/check_nginx.sh 16443"
}
 
vrrp\_instance VI\_1 { 
    state MASTER 
    interface eth0  # 修改为实际网卡名
    virtual\_router\_id 51 # VRRP 路由 ID实例，每个实例是唯一的 
    priority 100    # 优先级，备服务器设置 90 
    advert_int 1    # 指定VRRP 心跳包通告间隔时间，默认1秒 
    authentication { 
        auth_type PASS      
        auth_pass 1111 
    }  
    # 虚拟IP
    virtual_ipaddress { 
        192.168.1.199/24
    } 
    track_script {
        check_nginx
    } 
}
EOFCOPY
```
 
*   配置脚本
    
 
```
\[root@master1 ~\]# cat > /etc/keepalived/check_nginx.sh  << 'EOF'
#!/bin/bash
#keepalived 监控端口脚本
CHK_PORT=$1
if \[ -n "$CHK_PORT" \];then
        PORT\_PROCESS=\`ss -lnt|grep $CHK\_PORT|wc -l\`
        if \[ $PORT_PROCESS -eq 0 \];then
                echo "Port $CHK_PORT Is Not Used,End."
                exit 1
        fi
else
        echo "Check Port Cant Be Empty!"
fi
 
\[root@master1 ~\]# chmod +x  /etc/keepalived/check_nginx.shCOPY
```
 
### 
 
2、备keepalive
 
*   master2配置
    
 
```
\[root@master2 ~\]# cat > /etc/keepalived/keepalived.conf << 'EOF'
global_defs { 
   notification_email { 
     acassen@firewall.loc 
     failover@firewall.loc 
     sysadmin@firewall.loc 
   } 
   notification\_email\_from Alexandre.Cassen@firewall.loc  
   smtp_server 127.0.0.1 
   smtp\_connect\_timeout 30 
   router\_id NGINX\_BACKUP
} 
 
vrrp\_script check\_nginx {
    script "/etc/keepalived/check_nginx.sh 16443"
}
 
vrrp\_instance VI\_1 { 
    state BACKUP 
    interface eth0
    virtual\_router\_id 51 # VRRP 路由 ID实例，每个实例是唯一的 
    priority 90
    advert_int 1
    authentication { 
        auth_type PASS      
        auth_pass 1111 
    }  
    virtual_ipaddress { 
        192.168.1.199/24
    } 
    track_script {
        check_nginx
    } 
}
EOFCOPY
```
 
*   配置脚本
    
 
```
\[root@master2 ~\]# cat > /etc/keepalived/check_nginx.sh << 'EOF'
#!/bin/bash
#keepalived 监控端口脚本
CHK_PORT=$1
if \[ -n "$CHK_PORT" \];then
        PORT\_PROCESS=\`ss -lnt|grep $CHK\_PORT|wc -l\`
        if \[ $PORT_PROCESS -eq 0 \];then
                echo "Port $CHK_PORT Is Not Used,End."
                exit 1
        fi
else
        echo "Check Port Cant Be Empty!"
fi
 
\[root@master2 ~\]# chmod +x  /etc/keepalived/check_nginx.shCOPY
```
 
### 
 
3、启动服务
 
```
\[root@master1 ~\]# systemctl daemon-reload
\[root@master1 ~\]# systemctl start nginx
\[root@master1 ~\]# systemctl start keepalived
\[root@master1 ~\]# systemctl enable nginx keepalived
 
\[root@master2 ~\]# systemctl daemon-reload
\[root@master2 ~\]# systemctl start nginx
\[root@master2 ~\]# systemctl start keepalived
\[root@master2 ~\]# systemctl enable nginx keepalivedCOPY
```
 
### 
 
4、测试VIP是否绑定成功
 
```
\[root@master1 ~\]# ip a show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc pfifo\_fast state UP group default qlen 1000
    link/ether fa:37:df:7e:50:00 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.180/24 brd 192.168.1.255 scope global noprefixroute eth0
       valid\_lft forever preferred\_lft forever
    inet 192.168.1.199/24 scope global secondary eth0
       valid\_lft forever preferred\_lft forever
 
#停掉master1上的nginx。vip会漂移到master2
 \[root@master1 ~\]# systemctl stop nginx
 
 \[root@master2 ~\]# ip a show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc pfifo\_fast state UP group default qlen 1000
    link/ether fa:ef:7c:41:d7:00 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.181/24 brd 192.168.1.255 scope global noprefixroute eth0
       valid\_lft forever preferred\_lft forever
    inet 192.168.1.199/24 scope global secondary eth0COPY
```
 
### 
 
5、node修改vip连接
 
```
\[root@node1 ~\]# sed -i 's#192.168.1.160:6443#192.168.1.199:16443#' /opt/kubernetes/kubelet-bootstrap.kubeconfig
 
\[root@node1 ~\]# sed -i 's#192.168.1.160:6443#192.168.1.199:16443#' /opt/kubernetes/kubelet.kubeconfig
 
\[root@node1 ~\]# sed -i 's#192.168.1.160:6443#192.168.1.199:16443#' /opt/kubernetes/kube-proxy.kubeconfig
 
\[root@node1 ~\]# systemctl restart kubelet kube-proxyCOPY
```
 
### 
 
6、master修改vip连接
 
```
#kubectl
\[root@master1 ~\]#  sed -i 's#192.168.1.160:6443#192.168.1.199:16443#' /opt/kubernetes/kube.config
\[root@master1 ~\]#  sed -i 's#192.168.1.160:6443#192.168.1.199:16443#' /root/.kube/config
 
#kube-controller-manager
\[root@master1 ~\]#  sed -i 's#192.168.1.160:6443#192.168.1.199:16443#' /opt/kubernetes/kube-controller-manager.kubeconfig
\[root@master2 ~\]#  sed -i 's#192.168.1.160:6443#192.168.1.199:16443#' /opt/kubernetes/kube-controller-manager.kubeconfig
\[root@master3 ~\]#  sed -i 's#192.168.1.160:6443#192.168.1.199:16443#' /opt/kubernetes/kube-controller-manager.kubeconfig
 
#kube-scheduler
\[root@master1 ~\]#  sed -i 's#192.168.1.160:6443#192.168.1.199:16443#' /opt/kubernetes/kube-scheduler.kubeconfig
\[root@master2 ~\]#  sed -i 's#192.168.1.160:6443#192.168.1.199:16443#' /opt/kubernetes/kube-scheduler.kubeconfig
\[root@master3 ~\]#  sed -i 's#192.168.1.160:6443#192.168.1.199:16443#' /opt/kubernetes/kube-scheduler.kubeconfig
 
#重启服务
\[root@master1 ~\]# systemctl restart kube-controller-manager.service kube-scheduler.service 
\[root@master2 ~\]# systemctl restart kube-controller-manager.service kube-scheduler.service 
\[root@master3 ~\]# systemctl restart kube-controller-manager.service kube-scheduler.service COPY
```
 
七、更新k8s版本
 
 
-------------
 
操作步骤： 1、到nginx禁用master1配置 2、若master1有pod，先将master1设置维护状态，驱逐所有pod 3、下载新版本k8s，复制原k8s目录改名为新目录 4、复制并覆盖新组件服务至新k8s目录，更新软链接 5、重启相关服务，检查版本是否正常 6、取消维护状态，启用nginx配置 7、其他节点一样重复操作
 
```
#!/bin/bash
update_verison=v1.22.5
wget https://dl.k8s.io/$update_verison/kubernetes-server-linux-amd64.tar.gz
tar -xf kubernetes-server-linux-amd64.tar.gz 
cd kubernetes/server/bin/
mkdir /opt/kubernetes-$update_verison
cp -a /opt/kubernetes/* /opt/kubernetes-$update_verison/
\\cp -a kube-apiserver kube-controller-manager kube-scheduler kubectl /opt/kubernetes-$update_verison/
ln -snf /opt/kubernetes-$update_verison /opt/kubernetes 
systemctl restart kube-apiserver.service kube-scheduler.service kube-controller-manager.service
 
#node节点
node1: 
mkdir /opt/kubernetes-$update_verison
cp -a /opt/kubernetes/* /opt/kubernetes-$update_verison/
 
#!/bin/bash
cd kubernetes/server/bin/
\\cp -a kubelet kube-proxy /opt/kubernetes-$update_verison
ln -snf /opt/kubernetes-$update_verison /opt/kubernetes
systemctl restart kubelet kube-proxyCOPY
```
 
八、添加节点
 
 
----------
 
### 
 
1、添加node节点
 
*   新机器设置为node节点
    
 
```
```
 
 
#首先按照步骤一先初始化环境
mkdir -p /data/logs/kubernetes/{kubelet,kube-proxy}
rsync -aP node1:/opt/kubernetes* /opt/
rsync -aP node1:/usr/lib/systemd/system/{kube-proxy.service,kubelet.service} /usr/lib/systemd/system/
rm -f /opt/kubernetes/certs/kubelet-client-*
systemctl daemon-reload
systemctl enable --now kubelet kube-proxy
 
#在master节点批准请求访问
 
  
 
 
 
```
```
