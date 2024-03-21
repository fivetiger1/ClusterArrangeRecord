#    kubernetes Build
##   1. 集群部署说明
        集群部署使用二进制方式部署，主要分为master控制平台节点，node工作节点，以及持久化存储etcd节点，当然一般的，这些节点可以共用的，也就是在我们测试环境中可以存在一个master即是持久化存储etcd节点也可以是node工作节点，这个具体可以看资源的配置来调整。在生产中master和etcd在资源合理分配情况下可以共用，并且优先级是大于node（node可以临时停机检查，master和etcd非必要情况不停机）。在这里自用测试，我部署的配置为四台，共用以上节点身份。
##   2. 服务器配置说明
        自用测试：四台2核4G即可（后续使用ingress拉起pod较慢）
        生产环境：3台4核8G100~200SSD(master),3～5台4核8G100SSD（node）
##   3. 服务器初始化（所有节点）
        CentOS Linux release 7.9.2009 服务器创建完毕，网络桥接模式（由物理服务器ip段分配）  
###  3.1 配置静态ip
        # cat /etc/sysconfig/network-scripts/ifcfg-ens33
        TYPE="Ethernet"
        PROXY_METHOD="none"
        BROWSER_ONLY="no"
        BOOTPROTO="static"
        DEFROUTE="yes"
        NAME="ens33"
        UUID="5f1aa1f4-b4c2-4c77-b3ee-75211b9e0497"
        DEVICE="ens33"
        ONBOOT="yes"
        IPADDR=192.168.1.21
        NETMASK=255.255.255.0
        GATEWAY=192.168.1.1
###  3.2 修改主机名（永久修改），配置时间同步，免密登录
        # cat /etc/hostname
        master1
        # systemctl restart network

        # yum install ntpdate -y
        # cat /var/spool/cron/root
        */5 * * * * /usr/sbin/ntpdate ntp.ntsc.ac.cn
        
        # cat /etc/hosts
        127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
        ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
        192.168.1.11 master1
        192.168.1.12 master2
        192.168.1.21 node1
        192.168.1.22 node2
        # ssh-keygen  -t rsa -b 2048

        # list=(master1 master2 node1 node2)
        # for (( i=0;i<${#list[@]};i++ )) do ssh-copy-id ${list[i]};done
        # for (( i=0;i<${#list[@]};i++ )) do ssh ${list[i]} echo "${list[i]} success";done
####    linux服务器时间同步的意义在于计算机时钟会因为电源温度等因素进行微小偏移，但是长时间累积会造成误差超出服务预期，如事务处理，事件关联类型的服务，免密登录方便于后续k8s组件部署以及配置中通过hostname就可以解析到对应ip，直接进行通信，避开交互操作。在这里生成一次公私钥即可，其他机器拷贝该公钥，注意的是生成公私钥的节点也需要执行该步骤进行免密。
###  3.3 关闭防火墙，selinux，交换分区，ulimt限制修改
        # systemctl stop firewalld
        # systemctl disable firewalld
        # setenforce 0
        # sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
        # swapoff -a
        # cat /etc/fstab
        #/dev/mapper/centos-swap swap swap defaults 0 0
        # cat /etc/security/limits.conf
        *    soft    nofile    65536
        *    hard    nofile    65535
####    关闭防火墙，selinux都是为了方便集群内部通行，生产环境可以开放特定业务端口，而关闭swap分区是在于k8s在集群计算node节点内存资源时，不会计算交换分区，这样对于资源分配会产生误差影响集群服业务，因此需要关闭。而ulimt限制则是在centos系统默认的文件描述符限制太小，需要手动添加对应的配置，这属于业务服务器初始化中的常见步骤。
###  3.4 更新yum源
        # mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
        # wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
        # yum makecache
####    其他centos版本参考这个链接https://developer.aliyun.com/mirror/centos
###  3.5 模块加载，内核参数修改，ulimt配置修改
        modprobe br_netfilter
        lsmod |grep br_netfilter
####    br_netfilter模块可以使 iptables 规则可以在 Linux Bridges(二层)上面工作，用于将桥接的流量转发至iptables链，影响同node间的pod访问
        # cat  /etc/sysctl.d/k8s.conf
        net.ipv4.ip_forward=1 # 其值为0,说明禁止进行IP转发；如果是1,则说明IP转发功能已经打开。
        net.bridge.bridge-nf-call-ip6tables=1 # 是否在ip6tables链中过滤IPv6包
        net.bridge.bridge-nf-call-iptables=1 # 二层的网桥在转发包时也会被iptables的FORWARD规则所过滤，这样有时会出现L3层的iptables rules去过滤L2的帧的问题"
        
        # sysctl -p /etc/sysctl.d/k8s.conf
####    在这里从注解可以看出这步的修改是服务于集群网络部分，从最小单元pod出发，网络在namespace隔离基础上的通讯是由linux以太网桥（Linux Bridges） 实现，使用arp协议。具体是通过veth设备对一端连接pod网络命名空间，一端连接宿主机命名空间，这里命名空间也可以理解成网卡，相当于交换机的作用来解释目的地址的去向，从而进行通信。
##   4. 基础组件docker部署，镜像加速器配置（所有节点）
        # yum install -y yum-utils device-mapper-persistent-data lvm2
        # yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
        # sed -i 's+download.docker.com+mirrors.aliyun.com/docker-ce+' /etc/yum.repos.d/docker-ce.repo
        # yum makecache fast
        # yum -y install docker-ce
        # systemctl start docker && systemctl enable docker.service

        # cat /etc/docker/daemon.json
        { "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"] }
####    pod是k8s集群的最小单元，但是pod也是基于docker容器集成而来，也就是说pod中包含多个容器共同组成，并且k8s只是一个容器编排工具，实际跑服务的还是docker，包括镜像构建底层服务运行也是docker来实现的，不过对于庞大的k8s集群来说并不关心最底层服务的实现，在镜像构建环节已经敲定，更专注于服务的维护例如扩缩容，稳定，日志审计和监控方向，因此定位于最小pod单位即可。另外镜像加速器的作用在于docker默认会去海外拉取镜像，网路上会有问题，配置镜像加速器，也就是指定国内的镜像仓库进行拉取，实现加速的意义，但是在生产中，一般会创建自己的镜像仓库进行维护和使用。这里使用的是阿里云的镜像仓库，细节可以参考https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors
##   5. etc集群搭建，一般按照资源分配可以配置在master节点上，并且etcd节点至少三个节点，保证在单节点故障情况下数据的一致性和可用性(在这里使用自用测试环境我们部署在master1,master2,node1节点上)
###  5.1 证书工具部署 
        # cd /data/work 
        # wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 
        # wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 
        # wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64

        # chmod +x cfssl*
        # mv cfssl_linux-amd64 /usr/local/bin/cfssl
        # mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
        # mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo
###  5.2 证书创建
        # cat ca-csr.json
        {
            "CN": "etcd CA",
            "key": {
                "algo": "rsa",
                "size": 2048
            },
            "names": [
                {
                    "C": "CN",
                    "L": "Beijing",
                    "ST": "Beijing"
                }
            ]
        }
        # cfssl gencert -initca ca-csr.json | cfssljson -bare ca
        # cat ca-config.json
        {
            "signing": {
                "default": {
                    "expiry": "87600h"
                },
                "profiles": {
                    "www": {
                        "expiry": "87600h",
                        "usages": [
                            "signing",
                            "key encipherment",
                            "server auth",
                            "client auth"
                        ]
                    }
                }
            }
        }
        # cat  server-csr.json
        {
            "CN": "etcd",
            "hosts": [
            "192.168.1.11",
            "192.168.1.12",
            "192.168.1.21"
            ],
            "key": {
                "algo": "rsa",
                "size": 2048
            },
            "names": [
                {
                    "C": "CN",
                    "L": "BeiJing",
                    "ST": "BeiJing"
                }
            ]
        }
####    这里的hosts需要提前规划好，也就是你的etcd节点ip但是 生产环境要考虑扩容问题可以预留ip段
        # cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server
###  5.3 etcd集群部署  
        # cd /data/work/
        # wget https://github.com/etcd-io/etcd/releases/download/v3.4.13/etcd-v3.4.13- linux-amd64.tar.gz
        # tar xf etcd-v3.4.13-linux-amd64.tar.gz
        # cd etcd-v3.4.13-linux-amd64/
        # cp -p etcd-v3.4.13-linux-amd64/etcd* /usr/local/bin/
        # rsync -vaz etcd-v3.4.13-linux-amd64/etcd* master2:/usr/local/bin/
        # rsync -vaz etcd-v3.4.13-linux-amd64/etcd* node1:/usr/local/bin/
        # cat etcd.conf
        #[Member]
        ETCD_NAME="etcd-1"
        ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
        ETCD_LISTEN_PEER_URLS="https://192.168.1.11:2380"
        ETCD_LISTEN_CLIENT_URLS="https://192.168.1.11:2379"
        #[Clustering]
        ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.1.11:2380"
        ETCD_ADVERTISE_CLIENT_URLS="https://192.168.1.11:2379"
        ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.1.11:2380,etcd-2=https://192.168.1.12:2380,etcd-3=https://192.168.1.21:2380"
        ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
        ETCD_INITIAL_CLUSTER_STATE="new"
####    etcd.config 配置需要根据每台服务器具体ip进行修改，修改的字段有ETCD_NAME（往后排序即可），ETCD_LISTEN_PEER_URLS，ETCD_LISTEN_CLIENT_URLS，ETCD_INITIAL_ADVERTISE_PEER_URLS，ETCD_ADVERTISE_CLIENT_URLS
        # cat /lib/systemd/system/etcd.service
        [Unit]
        Description=Etcd Server
        After=network.target
        After=network-online.target
        Wants=network-online.target
        [Service]
        Type=notify
        EnvironmentFile=/opt/etcd/cfg/etcd.conf
        ExecStart=/opt/etcd/bin/etcd --cert-file=/opt/etcd/ssl/server.pem --key-file=/opt/etcd/ssl/server-key.pem --peer-cert-file=/opt/etcd/ssl/server.pem --peer-key-file=/opt/etcd/ssl/server-key.pem --trusted-ca-file=/opt/etcd/ssl/ca.pem --peer-trusted-ca-file=/opt/etcd/ssl/ca.pem --logger=zap
        Restart=on-failure
        LimitNOFILE=65536
        [Install]
        WantedBy=multi-user.target
        # cp ca*.pem /etc/etcd/ssl/ root@master1:/data/work
        # cp etcd*.pem /etc/etcd/ssl/ root@master2:/data/work
        # cp etcd.conf /etc/etcd/ root@node1:/data/work
        # for i in master1 master2 node1 ;do rsync -vaz etcd.conf $i:/etc/etcd/;done
        # for i in master1 master2 node1 ;do rsync -vaz etcd*.pem ca*.pem $i:/etc/etcd/ssl/;done
        # for i in master1 master2 node1 ;do rsync -vaz etcd.service $i:/lib/systemd/system/;done
        # systemctl daemon-reload
        # systemctl enable etcd
        # systemctl start etcd
        #  /opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://192.168.1.11:2379,https://192.168.1.12:2379,https://192.168.1.21:2379" endpoint health
        https://192.168.1.21:2379 is healthy: successfully committed proposal: took = 12.799202ms
        https://192.168.1.11:2379 is healthy: successfully committed proposal: took = 16.710788ms
        https://192.168.1.12:2379 is healthy: successfully committed proposal: took = 18.935425ms