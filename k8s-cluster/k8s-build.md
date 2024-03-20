#   kubernetes Build
##  1.集群部署说明
      集群部署主要分为master控制平台节点，node工作节点，以及持久化存储etcd节点，当然一般的，这些节点可以共用的，也就是在我们测试环境中可以存在一个master即是持久化存储etcd节点也可以是node工作节点，这个具体可以看资源的配置来调整。在生产中master和etcd在资源合理分配情况下可以共用，并且优先级是大于node（node可以临时停机检查，master和etcd非必要情况不停机）。在这里自用测试，我部署的配置为四台，共用以上节点身份。
##  2.服务器配置说明
      自用测试：四台2核4G即可（后续使用ingress拉起pod较慢）
      生产环境：3台4核8G100~200SSD(master),3～5台4核8G100SSD（node）
##  3.服务器初始化
      CentOS Linux release 7.9.2009 服务器创建完毕，网络桥接模式（由物理服务器ip段分配）  
####  1.配置静态ip
      cat /etc/sysconfig/network-scripts/ifcfg-ens33
      
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