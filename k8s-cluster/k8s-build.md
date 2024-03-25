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
####    linux服务器时间同步的意义在于计算机时钟会因为电源温度等因素进行微小偏移，但是长时间累积会造成误差超出服务预期，如事务处理，事件关联类型的服务，免密登录方便于后续k8s组件部署以及配置中通过hostname就可以解析到对应ip，直接进行通信，避开交互操作，另外免密登录可以简化服务账户的配置，使得Pod能够无需密码地访问API服务器。在这里生成一次公私钥即可，其他机器拷贝该公钥，注意的是生成公私钥的节点也需要执行该步骤进行免密。
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
        # modprobe -- ip_vs 
        # modprobe -- ip_vs_rr
        # modprobe -- ip_vs_wrr 
        # modprobe -- ip_vs_sh
        # modprobe -- nf_conntrack_ipv4
        # lsmod | grep ip_vs
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
##   5. etcd集群搭建，一般按照资源分配可以配置在master节点上，并且etcd节点至少三个节点，保证在单节点故障情况下数据的一致性和可用性(在这里使用自用测试环境我们部署在master1,master2,node1节点上)
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
        # cat /data/TLS/etcd/ca-csr.json
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

        # cat /data/TLS/etcd/ca-config.json
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

        # cat  /data/TLS/etcd/server-csr.json
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
####    etcd分布式的键值存储系统 保证数据可用性一致性解释。是通过Raft协议实现，该协议是分布式系统中的复制日志的一致性算法。可用性他通过任期的唯一序列号跟踪时间，通过心跳检测确认leader节点是否异常，多数派（超过一半的节点）推选出领导者（具体的推选过程follower将自己任期号加一发送其他节点投票消息，其他follower节点对比自己任期号和接收到的任期号晚，会更新心跳检测周期也就是选举计时器，并投票返回发送方同意信息，这个过程中可能需要几轮投票后选举出来，遵循多数派选举），从而保证领导者的日志条目复制确认提交任务的正常发送。而一致性是通过日志复制来实现，具体的过程是客户端写入请求后，leader创建新的日志条目，并行发送其他follower AppendEntries请求（包含日志条目数据，leader接待你任期信息，以及上一个日志条目的索引和值），follower节点复制该日志条目后检查有效性（任期序列号是最新的，上一个日志条目索引和值是正确的）并添加到自己日志中返回leader一个AppendEntries响应，leader收到了大多数确认响应后，做标记committed，并更新etcd存储的数据，同时发送其他follower日志条目已提交条目，follower 更新etcd存储的数据，进而保证了数据的一致性。
####    etcd foller节点故障恢复后数据处理，在重新加入集群后，启动追赶机制，通过日志压缩（发送AppendEntries请求获取缺失日志条目），快照（在日志落后多的情况下，包含集群当前状态的一致性视图），日志复制（获取快照之后的缺失日志条目）的方式，这追赶过程中，leader和其他follwer确保该问题节点的一致性，并且在过程中不会处理客户端请求直到问题节点提交被大多数节点复制并同意的日志条目。

##   6. Kubernetes组件部署
####    Kubernetes 组件调用过程，k8s主要是对pod资源进行编排管理，那么这个过程其实也就是组件的调用过程，首先我们通过kubectl调用apiserver，发送资源管理信息，apiserver会将本次调用信息储存在etcd中，接着scheduler通过各个工作节点汇报过来的资源使用情况(包括pod资源需求，亲和性规则，污点和容忍度策略)对本次调用pod进行资源分配，并使用apiserver 存储在etcd中更新pod状态为调度中，工作节点的kubectl通过kube-proxy通信实时获取apiserver到调用任务，对pod资源进行更新创建删除，在创建pod后网络插件会为pod分配ip地址设置网络策略，确保pod与集群其他pod能进行通信，后续kubectl 会定期向apiserver汇报pod和容器状态。
####    Kubernetes 亲和性规则，污点和容忍度解释。亲和性分为节点亲和和pod亲和，节点亲和也就是可以通过节点标签指定pod部署的节点。而pod亲和则是指pod之前的亲和性和范亲和性，这两种逆反的规则是也是通过pod标签方便相关pod在同一节点，方便通信和维护，另外就是为了使对应pod不在同一节点上 避免单点故障。而污点是指在节点层面的标记，意味被标记节点对pod部署的许可，而容忍度则是在这个基础上，匹配到了容忍度依然可以在已标记污点的节点上进行pod部署，这两个机制可以用于pod驱逐或者，关键服务节点。
###  6.1 组件证书生成
        # cat /data/TLS/k8s/ca-config.json
        {
            "signing": {
                "default": {
                    "expiry": "87600h"
                },
                "profiles": {
                    "kubernetes": {
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

        # cat /data/TLS/k8s/ca-csr.json
        {
            "CN": "kubernetes",
            "key": {
                "algo": "rsa",
                "size": 2048
            },
            "names": [
                {
                    "C": "CN",
                    "L": "Beijing",
                    "ST": "Beijing",
                    "O": "k8s",
                    "OU": "System"
                }
            ]
        }

        # cfssl gencert -initca ca-csr.json | cfssljson -bare ca -

        # cat /data/TLS/k8s/server-csr.json
        {
            "CN": "kubernetes",
            "hosts": [
            "192.168.0.1",
            "127.0.0.1",
            "192.168.1.8",
            "192.168.1.11",
            "192.168.1.12",
            "192.168.1.13",
            "192.168.1.21",
            "192.168.1.22",
            "192.168.1.23",
            "192.168.1.24",
            "192.168.1.25",
            "192.168.1.26",
            "192.168.1.27",
            "kubernetes",
            "kubernetes.default",
            "kubernetes.default.svc",
            "kubernetes.default.svc.cluster",
            "kubernetes.default.svc.cluster.local"
            ],
            "key": {
                "algo": "rsa",
                "size": 2048
            },
            "names": [
                {
                    "C": "CN",
                    "L": "BeiJing",
                    "ST": "BeiJing",
                    "O": "k8s",
                    "OU": "System"
                }
            ]
        }
####    这里也是使用该证书的host,也就是你的k8s集群节点，生产环境要考虑扩容问题可以预留ip段
        # cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server
        
        # cat /data/TLS/k8s/kube-proxy-csr.json
        {
        "CN": "system:kube-proxy",
        "hosts": [],
        "key": {
            "algo": "rsa",
            "size": 2048
        },
        "names": [
            {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing",
            "O": "k8s",
            "OU": "System"
            }
        ]
        }

        # cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
        
        # cat /data/TLS/k8s/admin-csr.json
        {
        "CN": "admin",
        "hosts": [],
        "key": {
            "algo": "rsa",
            "size": 2048
        },
        "names": [
            {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing",
            "O": "system:masters",
            "OU": "System"
            }
        ]
        }
        
        # cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
        # mkdir /opt/kubernetes/{bin,cfg,ssl,logs} -p
        # cp ca.pem ca-key.pem server.pem server-key.pem /opt/kubernetes/ssl/
        # cd /opt/kubernetes/cfg/
        # head -c 16 /dev/urandom | od -An -t x | tr -d ' '
        # cat token.csv
        3ca482ccfaf101b8f177679c5e36112c,kubelet-bootstrap,10001,"system:nodebootstrapper"
###  6.2 master 组件部署 下载地址 https://github.com/kubernetes/kubernetes/tree/master/CHANGELOG 下载后解压并进入解压后的目录
        # mkdir /opt/kubernetes/{bin,cfg,ssl,logs} -p
        # cd kubernetes/server/bin/
        # cp kube-apiserver kube-scheduler kube-controller-manager kubectl  /opt/kubernetes/bin/
        # rsync -vaz kube-apiserver kube- controller-manager kube-scheduler kubectl master2:/opt/kubernetes/bin/

        # for i in master1 master2 node1 node2 ;do rsync -vaz kubelet kube-proxy $i:/opt/kubernetes/bin/;done

        # cat /opt/kubernetes/cfg/kube-apiserver.conf
        KUBE_APISERVER_OPTS="--logtostderr=false \
        --v=2 \
        --log-dir=/opt/kubernetes/logs \
        --etcd-servers=https://192.168.1.11:2379,https://192.168.1.12:2379,https://192.168.1.21:2379 \
        --bind-address=192.168.1.11 \
        --secure-port=6443 \
        --advertise-address=192.168.1.11 \
        --allow-privileged=true \
        --service-cluster-ip-range=192.168.0.0/24 \
        --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \
        --authorization-mode=RBAC,Node \
        --enable-bootstrap-token-auth=true \
        --token-auth-file=/opt/kubernetes/cfg/token.csv \
        --service-node-port-range=30000-32767 \
        --kubelet-client-certificate=/opt/kubernetes/ssl/server.pem \
        --kubelet-client-key=/opt/kubernetes/ssl/server-key.pem \
        --tls-cert-file=/opt/kubernetes/ssl/server.pem  \
        --tls-private-key-file=/opt/kubernetes/ssl/server-key.pem \
        --client-ca-file=/opt/kubernetes/ssl/ca.pem \
        --service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \
        --service-account-issuer=api \
        --service-account-signing-key-file=/opt/kubernetes/ssl/server-key.pem \
        --etcd-cafile=/opt/etcd/ssl/ca.pem \
        --etcd-certfile=/opt/etcd/ssl/server.pem \
        --etcd-keyfile=/opt/etcd/ssl/server-key.pem \
        --requestheader-client-ca-file=/opt/kubernetes/ssl/ca.pem \
        --proxy-client-cert-file=/opt/kubernetes/ssl/server.pem \
        --proxy-client-key-file=/opt/kubernetes/ssl/server-key.pem \
        --requestheader-allowed-names=kubernetes \
        --requestheader-extra-headers-prefix=X-Remote-Extra- \
        --requestheader-group-headers=X-Remote-Group \
        --requestheader-username-headers=X-Remote-User \
        --enable-aggregator-routing=true \
        --audit-log-maxage=30 \
        --audit-log-maxbackup=3 \
        --audit-log-maxsize=100 \
        --feature-gates=RemoveSelfLink=false \
        --audit-log-path=/opt/kubernetes/logs/k8s-audit.log"
####    这里根据实际节点ip进行修改，需要修改的字段有 --bind-address，--advertise-address，需要注意的是 --service-cluster-ip-range 这个字段就是未来你创建service的ip段
        # cat /usr/lib/systemd/system/kube-apiserver.service
        [Unit]
        Description=Kubernetes API Server
        Documentation=https://github.com/kubernetes/kubernetes

        [Service]
        EnvironmentFile=/opt/kubernetes/cfg/kube-apiserver.conf
        ExecStart=/opt/kubernetes/bin/kube-apiserver $KUBE_APISERVER_OPTS
        Restart=on-failure

        [Install]
        WantedBy=multi-user.target
        
        # systemctl daemon-reload
        # systemctl enable kube-apiserver
        # systemctl restart kube-apiserver
        # systemctl status kube-apiserver

        # cat /opt/kubernetes/cfg/kube-controller-manager.conf
        KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=false \
        --v=2 \
        --log-dir=/opt/kubernetes/logs \
        --leader-elect=true \
        --kubeconfig=/opt/kubernetes/cfg/kube-controller-manager.kubeconfig \
        --bind-address=127.0.0.1 \
        --allocate-node-cidrs=true \
        --cluster-cidr=192.244.0.0/16 \
        --service-cluster-ip-range=192.168.0.0/24 \
        --cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \
        --cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem  \
        --root-ca-file=/opt/kubernetes/ssl/ca.pem \
        --service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \
        --cluster-signing-duration=87600h0m0s"
####    这里需要注意的是--cluster-cidr，该字段指定集群的 Pod 网络 CIDR 范围。这是集群中所有 Pod 的 IP 地址范围

        # cat /usr/lib/systemd/system/kube-controller-manager.service
        [Unit]
        Description=Kubernetes Controller Manager
        Documentation=https://github.com/kubernetes/kubernetes
        [Service]
        EnvironmentFile=/opt/kubernetes/cfg/kube-controller-manager.conf
        ExecStart=/opt/kubernetes/bin/kube-controller-manager $KUBE_CONTROLLER_MANAGER_OPTS
        Restart=on-failure
        [Install]
        WantedBy=multi-user.target

        # systemctl daemon-reload
        # systemctl enable kube-controller-manager
        # systemctl restart kube-controller-manager
        # systemctl status kube-controller-manager

        # cat /opt/kubernetes/cfg/kube-scheduler.conf
        KUBE_SCHEDULER_OPTS="--logtostderr=false \
        --v=2 \
        --log-dir=/opt/kubernetes/logs \
        --leader-elect \
        --kubeconfig=/opt/kubernetes/cfg/kube-scheduler.kubeconfig \
        --bind-address=127.0.0.1"

        # cat /usr/lib/systemd/system/kube-scheduler.service
        [Unit]
        Description=Kubernetes Scheduler
        Documentation=https://github.com/kubernetes/kubernetes
        [Service]
        EnvironmentFile=/opt/kubernetes/cfg/kube-scheduler.conf
        ExecStart=/opt/kubernetes/bin/kube-scheduler $KUBE_SCHEDULER_OPTS
        Restart=on-failure
        [Install]
        WantedBy=multi-user.target

        # systemctl daemon-reload
        # systemctl enable kube-scheduler
        # systemctl restart kube-scheduler
        # systemctl status kube-scheduler

        # kubectl create clusterrolebinding kubelet-bootstrap \
        --clusterrole=system:node-bootstrapper \
        --user=kubelet-bootstrap
        # kubectl get cs
        Warning: v1 ComponentStatus is deprecated in v1.19+
        NAME                 STATUS    MESSAGE             ERROR
        scheduler            Healthy   ok                  
        controller-manager   Healthy   ok                  
        etcd-2               Healthy   {"health":"true"}   
        etcd-0               Healthy   {"health":"true"}   
        etcd-1               Healthy   {"health":"true"}
###  6.2 node 组件部署  
        # mkdir /opt/kubernetes/{bin,cfg,ssl,logs} -p
        # cat /opt/kubernetes/cfg/kubelet.conf
        KUBELET_OPTS="--logtostderr=false \
        --v=2 \
        --log-dir=/opt/kubernetes/logs \
        --hostname-override=master1 \
        --network-plugin=cni \
        --kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \
        --bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \
        --config=/opt/kubernetes/cfg/kubelet-config.yml \
        --cert-dir=/opt/kubernetes/ssl \
        --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause-amd64:3.0"
####    这里需要注意 --hostname-override 修改成主机名
        # cat /opt/kubernetes/cfg/kubelet-config.yml
        kind: KubeletConfiguration
        apiVersion: kubelet.config.k8s.io/v1beta1
        address: 0.0.0.0
        port: 10250
        readOnlyPort: 10255
        cgroupDriver: cgroupfs
        clusterDNS:
        - 192.168.0.2
        clusterDomain: cluster.local 
        failSwapOn: false
        authentication:
        anonymous:
            enabled: false
        webhook:
            cacheTTL: 2m0s
            enabled: true
        x509:
            clientCAFile: /opt/kubernetes/ssl/ca.pem 
        authorization:
        mode: Webhook
        webhook:
            cacheAuthorizedTTL: 5m0s
            cacheUnauthorizedTTL: 30s
        evictionHard:
        imagefs.available: 15%
        memory.available: 100Mi
        nodefs.available: 10%
        nodefs.inodesFree: 5%
        maxOpenFiles: 1000000
        maxPods: 110
####    这里是kubelet的配置说明，也就是对节点资源汇报的说明，例如evictionHard:：当节点资源达到这些阈值时，Kubelet将开始驱逐Pods。imagefs.available: 15%：节点上可用的镜像文件系统（imagefs）空间至少为15%。memory.available: 100Mi：节点上可用的内存至少为100Mi（100兆字节）。nodefs.available: 10%：节点文件系统（nodefs）可用空间至少为10%。nodefs.inodesFree: 5%：节点文件系统可用的inode数量至少为5%
        # cd /opt/kubernetes/ssl
        # cp /root/kubernetes/server/bin/kubectl /usr/bin
        # KUBE_APISERVER="https://192.168.1.11:6443"
        # TOKEN="3ca482ccfaf101b8f177679c5e36112c"
        # kubectl config set-cluster kubernetes \
        --certificate-authority=/opt/kubernetes/ssl/ca.pem \
        --embed-certs=true \
        --server=${KUBE_APISERVER} \
        --kubeconfig=bootstrap.kubeconfig
        # kubectl config set-credentials "kubelet-bootstrap" \
        --token=${TOKEN} \
        --kubeconfig=bootstrap.kubeconfig
        # kubectl config set-context default \
        --cluster=kubernetes \
        --user="kubelet-bootstrap" \
        --kubeconfig=bootstrap.kubeconfig
        # kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
        # cp bootstrap.kubeconfig /opt/kubernetes/cfg
        # cat /usr/lib/systemd/system/kubelet.service
        [Unit]
        Description=Kubernetes Kubelet
        After=docker.service

        [Service]
        EnvironmentFile=/opt/kubernetes/cfg/kubelet.conf
        ExecStart=/opt/kubernetes/bin/kubelet $KUBELET_OPTS
        Restart=on-failure
        LimitNOFILE=65536

        [Install]
        WantedBy=multi-user.target
        # systemctl daemon-reload
        # systemctl start kubelet
        # systemctl enable kubelet
        # systemctl status kubelet
        
        # kubectl get csr
        # kubectl certificate approve

        # cat /opt/kubernetes/cfg/kube-proxy.conf
        KUBE_PROXY_OPTS="--logtostderr=false \
        --v=2 \
        --log-dir=/opt/kubernetes/logs \
        --config=/opt/kubernetes/cfg/kube-proxy-config.yml"
        # cat /opt/kubernetes/cfg/kube-proxy-config.yml
        kind: KubeProxyConfiguration
        apiVersion: kubeproxy.config.k8s.io/v1alpha1
        bindAddress: 0.0.0.0
        metricsBindAddress: 0.0.0.0:10249
        clientConnection:
        kubeconfig: /opt/kubernetes/cfg/kube-proxy.kubeconfig
        hostnameOverride: master1
        clusterCIDR: 192.168.0.0/24

        # KUBE_APISERVER="https://192.168.71.11:6443"
        # kubectl config set-cluster kubernetes  --certificate-authority=/opt/kubernetes/ssl/ca.pem  --embed-certs=true  --server=${KUBE_APISERVER} --kubeconfig=kube-proxy.kubeconfig
        # kubectl config set-credentials kube-proxy  --client-certificate=./kube-proxy.pem  --client-key=./kube-proxy-key.pem  --embed-certs=true  --kubeconfig=kube-proxy.kubeconfig
        # kubectl config set-context default  --cluster=kubernetes  --user=kube-proxy  --kubeconfig=kube-proxy.kubeconfig
        # kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
        # cp kube-proxy.kubeconfig /opt/kubernetes/cfg/
        
        # cat /usr/lib/systemd/system/kube-proxy.service
        [Unit]
        Description=Kubernetes Proxy
        After=network.target

        [Service]
        EnvironmentFile=/opt/kubernetes/cfg/kube-proxy.conf
        ExecStart=/opt/kubernetes/bin/kube-proxy $KUBE_PROXY_OPTS
        Restart=on-failure
        LimitNOFILE=65536

        [Install]
        WantedBy=multi-user.target
        # systemctl daemon-reload 
        # systemctl start kube-proxy 
        # systemctl enable kube-proxy
        # systemctl status kube-proxy
###  6.3 部署cni网络插件
        # wget https://docs.projectcalico.org/v3.14/manifests/calico.yaml --no-check- certificate
        # kubectl apply -f calico.yaml
###  6.4 部署coredns 
        # wget https://raw.githubusercontent.com/coredns/deployment/master/kubernetes/coredns.yaml.sed
        # mv coredns.yaml.sed coredns.yaml
####    这里需要修改 搜索 clusterIP 修改（kubelet配置文件中的clusterDNS） 搜索ready替换下一行以kubernetes开头的文本kubernetes cluster.local in-addr.arpa ip6.arpa
        # kubectl apply -f coredns.yaml
##   7. Kubernetes负载均衡和高可用部署
###  7.1 nginx部署 （在master节点上）下载这里就不再赘述 仅展示配置文件
        user nginx;
        worker_processes auto;
        error_log /usr/local/nginx/logs/error.log;
        pid /usr/local/nginx/logs/nginx.pid;
        include /usr/share/nginx/modules/*.conf;
        events {
            worker_connections 1024;
        }
        # 四层负载均衡，为两台Master apiserver组件提供负载均衡
        stream {
            log_format  main  '$remote_addr $upstream_addr - [$time_local] $status $upstream_bytes_sent';
            access_log  /usr/local/nginx/logs/k8s-access.log  main;
            upstream k8s-apiserver {
            server 192.168.1.11:6443;   # Master1 APISERVER IP:PORT
            server 192.168.1.12:6443;   # Master2 APISERVER IP:PORT
            }
            server {
            listen 16443; # 注意如果nginx与master节点复用，这个监听端口不能是6443，否则会冲突
            proxy_pass k8s-apiserver;
            }
        }
        http {
            log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                            '$status $body_bytes_sent "$http_referer" '
                            '"$http_user_agent" "$http_x_forwarded_for"';
            access_log  /usr/local/nginx/logs/access.log  main;
            sendfile            on;
            tcp_nopush          on;
            tcp_nodelay         on;
            keepalive_timeout   65;
            types_hash_max_size 2048;
            include             /usr/local/nginx/mime.types;
            default_type        application/octet-stream;
            server {
                listen       80 default_server;
                server_name  _;
                location / {
                }
            }
        }
###  7.2 keepalived部署 （在master节点上）下载这里就不再赘述 仅展示配置文件
        global_defs { 
        notification_email { 
            acassen@firewall.loc 
            failover@firewall.loc 
            sysadmin@firewall.loc 
        } 
        notification_email_from Alexandre.Cassen@firewall.loc  
        smtp_server 127.0.0.1 
        smtp_connect_timeout 30 
        router_id NGINX_MASTER
        } 
        vrrp_script check_nginx {
            script "/etc/keepalived/check_nginx.sh"
        }
        vrrp_instance VI_1 { 
            state MASTER 
            interface ens33  # 修改为实际网卡名
            virtual_router_id 51 # VRRP 路由 ID实例，每个实例是唯一的 
            priority 100    # 优先级，备服务器设置 90 
            advert_int 1    # 指定VRRP 心跳包通告间隔时间，默认1秒 
            authentication { 
                auth_type PASS      
                auth_pass 1111 
            }  
            # 虚拟IP
            virtual_ipaddress { 
                192.168.1.8/24
            } 
            track_script {
                check_nginx
            } 
        }
        # cat check_nginx.sh
        #!/bin/bash
        count=$(ss -antp |grep 16443 |egrep -cv "grep|$$")
        ​
        if [ "$count" -eq 0 ];then
            exit 1
        else
            exit 0
        fi