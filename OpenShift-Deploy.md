```textile
作者：李晓辉

微信：Lxh_Chat

邮箱：xiaohui_li@foxmail.com
```

| 角色                                             | IP            | 主机名                           | CPU   | 内存  | 操作系统       |
| ---------------------------------------------- | ------------- | ----------------------------- | ----- | --- | ---------- |
| DHCP<br>NTP<br>DNS<br>HAProxy<br>TFTP<br>HTTPD | 192.168.8.200 | content.cluster1.xiaohui.cn   | 4-8核心 | 8G  | RHEL9.5 |
| bootstrap                                      | 192.168.8.201 | bootstrap.cluster1.xiaohui.cn | 4-8核心 | 16G |rhcos            |
| master                                         | 192.168.8.202 | master1.cluster1.xiaohui.cn   | 4-8核心 | 16G |rhcos            |
| master                                         | 192.168.8.203 | master1.cluster1.xiaohui.cn   | 4-8核心 | 16G |rhcos            |
| master                                         | 192.168.8.204 | master2.cluster1.xiaohui.cn   | 4-8核心 | 16G |rhcos            |
| compute0                                       | 192.168.8.205 | compute0.cluster1.xiaohui.cn  | 4-8核心 | 16G |rhcos            |
| compute1                                       | 192.168.8.206 | compute1.cluster1.xiaohui.cn  | 4-8核心 | 16G |rhcos            |

这篇文章介绍了UPI的方式来部署OpenShift，其中涉及到DHCP、TFTP、DNS、HTTP、HAProxy等基础服务的部署和配置，过程较为复杂，每个服务在部署后，都尽量检查一下是否正常工作，再进行下一步的服务部署，避免中间有遗漏造成效果达不到的情况。

# 配置DHCP

本次部署采用了DHCP的方式来部署Bootstrap、Master、worker等节点，而且还有一个好处就是不用在每个节点初始部署时，手工输入一大串麻烦的网络信息以及coreos的各种参数，我这里的DHCP直接给这写节点分配固定的IP地址，然后配合PXE的配置文件，就可以省却手工在每台节点输入的事，非常方便，还提高了正确性，且易于重用。

**以下我的的DHCP配置概况：**

1. DHCP工作在192.168.8.0/24子网中
2. 对外提供的IP为192.168.8.201-192.168.8.250
3. 对外提供的DNS为192.168.8.200
4. 对外提供的网关为192.168.8.2
5. 根据DHCP客户端提供的厂商类别识别代码来判断应该用bios还是uefi提供服务
6. 客户端拿到上述网络信息后，以TFTP协议连接的地址为192.168.8.200，去拿pxelinux.0或grubx64.efi引导文件

在此之前，我配置好了软件仓库，你可以考虑从RHEL9.5的ISO安装软件，这需要你配置yum仓库

```bash
yum install dhcp-server -y
```

1. 配置文件这里要注意subnet下面的几行，这是我们的工作子网信息与分发地址的信息

2. 下面我为节点做了dhcp地址的预留，这里要注意mac地址必须要和将来的机器保持一致


```bash
cat > /etc/dhcp/dhcpd.conf <<EOF
option space PXE;
option arch code 93 = unsigned integer 16; # RFC4578
subnet 192.168.8.0 netmask 255.255.255.0 {
  range 192.168.8.201 192.168.8.2;
  option routers 192.168.8.2;
  option domain-name-servers 192.168.8.200;
  default-lease-time 600;
  max-lease-time 7200;
class "pxeclients" {
  match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
  next-server 192.168.8.200;
  if option arch = 00:07 {
      filename "grubx64.efi";
  } else if option arch = 00:08 {
      filename "grubx64.efi";
  } else if option arch = 00:09 {
      filename "grubx64.efi";
  } else {
      filename "pxelinux.0";
  }
  }
}
host bootstrap {
  hardware ethernet 00:50:56:94:8a:21;
  fixed-address 192.168.8.201;
  server-name "bootstrap.cluster1.xiaohui.cn";
}
host master0 {
  hardware ethernet 00:50:56:94:07:6d;
  fixed-address 192.168.8.202;
  server-name "master0.cluster1.xiaohui.cn";
}
host master1 {
  hardware ethernet 00:50:56:94:5c:7d;
  fixed-address 192.168.8.203;
  server-name "master1.cluster1.xiaohui.cn";
}
host master2 {
  hardware ethernet 00:50:56:94:b6:21;
  fixed-address 192.168.8.204;
  server-name "master2.cluster1.xiaohui.cn";
}
host compute0 {
  hardware ethernet 00:50:56:94:ce:83;
  fixed-address 192.168.8.205;
  server-name "compute0.cluster1.xiaohui.cn";
}
host compute1 {
  hardware ethernet 00:50:56:94:fd:5a;
  fixed-address 192.168.8.206;
  server-name "compute1.cluster1.xiaohui.cn";
} 
EOF
```

不要忘记启动dhcpd服务以及开通dhcp防火墙

```bash
systemctl enable dhcpd --now
firewall-cmd --add-service=dhcp --permanent
firewall-cmd --reload
```

# 配置NTP

 配置NTP时间同步可以确保时间的准确性，这在集群中，是非常重要的，此处将ntp.aliyun.com作为content机器的上游时间源，并将自己作为时间源，向192.168.8.0/24网段进行授时

```bash
yum install chrony -y
sed -i '/^pool/a\pool ntp.aliyun.com iburst' /etc/chrony.conf
sed -i '/^# allow/a\allow 192.168.8.0/24' /etc/chrony.conf
systemctl enable chronyd --now
firewall-cmd --add-service=ntp --permanent
firewall-cmd --reload
```

# 配置HTTPD

在PXE过程中，需要从网络中下载rootfs以及各个节点配置文件，我们部署一个httpd的网站服务，并从2000端口提供配置文件等内容，内容规划放在/content中，这里要注意不要使用默认的80端口，因为稍后我们的HAProxy将工作在80端口为OpenShift正常提供Ingress服务

```bash
yum install httpd policycoreutils-python-utils -y
semanage port -m -t http_port_t -p tcp 2000
mkdir /content
sed -i 's/Listen 80/Listen 2000/g' /etc/httpd/conf/httpd.conf
```

```bash
cat > /etc/httpd/conf.d/customsite.conf << 'EOF'
<VirtualHost *:2000>
    ServerName content.cluster.xiaohui.cn
    DocumentRoot "/content"
</VirtualHost>
<Directory "/content">
<RequireAll>
    Require all granted
</RequireAll>
</Directory>
EOF
```

```bash
firewall-cmd --add-port=2000/tcp --permanent
firewall-cmd --reload
systemctl enable httpd --now
```

# 配置DNS

DNS条目在openshift安装过程中承担了很重要的角色，我们的域名为xiaohui.cn，集群名称预备设置为cluster1，反向记录很重要，虽然我们有DHCP绑定并分配名称，但PTR可以直接设置主机名，在后续任务中，承担非常重要的角色，例如，采用网页工具安装，发现主机页面的主机名，就是通过PTR记录获得的

| DNS                           | 角色         | 功能                                    |
| ----------------------------- | ---------- | ------------------------------------- |
| content.cluster1.xiaohui.cn   | material   | 提供DHCP、HTTP、NTP、HTTP、DNS、TFTP、HAProxy |
| api.cluster1.xiaohui.cn       | master api | 提供openshift master api接口，通常指向负载均衡     |
| api-int.cluster1.xiaohui.cn   | master api | 提供api的内部通信，通常指向负载均衡                   |
| *.apps.cluster1.xiaohui.cn    | APP        | 应用程序通配符，用于后期集群内的应用暴露，通常指向负载均衡         |
| bootstrap.cluster1.xiaohui.cn | bootstrap  | 用于准备集群，集群就绪后可删除                       |
| masterX.cluster1.xiaohui.cn   | master     | openshift master服务                    |
| computeX.cluster1.xiaohui.cn  | worker     | openshift 计算节点服务                      |


在DNS的配置文件中，我们工作在本机的所有IP地址上，并允许所有人来查询DNS记录，同时开启了转发器，有其他查询时，发给114.114.114.114，关闭了dnssec验证

```bash
yum install bind bind-utils -y

sed -i 's/listen-on port 53 { 127.0.0.1; }/listen-on port 53 { any; }/g' /etc/named.conf
sed -i 's/allow-query     { localhost; };/allow-query     { any; };/g' /etc/named.conf
sed -i '/recursion yes;/a\        forward first;\n        forwarders { 114.114.114.114; };' /etc/named.conf
sed -i 's/dnssec-validation yes;/dnssec-validation no;/' /etc/named.conf
```

这里我们为cluster1.xiaohui.cn域名提供了正向和反向解析

```bash
cat >> /etc/named.rfc1912.zones <<EOF
zone "cluster1.xiaohui.cn" IN {
        type master;
        file "cluster1.xiaohui.cn.zone";
};

zone "8.168.192.in-addr.arpa" IN {
        type master;
        file "cluster1.xiaohui.cn.ptr";
};
EOF
```

以下是区域文件中具体的解析内容

```bash
cat > /var/named/cluster1.xiaohui.cn.zone <<'EOF'
$TTL 1W
@       IN      SOA     ns1.cluster1.xiaohui.cn. root.cluster1.xiaohui.cn (
                        1               ; serial
                        3H              ; refresh (3 hours)
                        30M             ; retry (30 minutes)
                        2W              ; expiry (2 weeks)
                        1W )            ; minimum (1 week)
        IN      NS      ns1.cluster1.xiaohui.cn.
ns1.cluster1.xiaohui.cn.         IN      A       192.168.8.200
content.cluster1.xiaohui.cn.     IN      A       192.168.8.200
api.cluster1.xiaohui.cn.         IN      A       192.168.8.200
api-int.cluster1.xiaohui.cn.     IN      A       192.168.8.200
*.apps.cluster1.xiaohui.cn.      IN      A       192.168.8.200
bootstrap.cluster1.xiaohui.cn.   IN      A       192.168.8.201
master0.cluster1.xiaohui.cn.     IN      A       192.168.8.202
master1.cluster1.xiaohui.cn.     IN      A       192.168.8.203
master2.cluster1.xiaohui.cn.     IN      A       192.168.8.204
compute0.cluster1.xiaohui.cn.    IN      A       192.168.8.205
compute1.cluster1.xiaohui.cn.    IN      A       192.168.8.206
EOF


cat > /var/named/cluster1.xiaohui.cn.ptr <<'EOF'
$TTL 1W
@       IN      SOA     ns1.cluster1.xiaohui.cn. root.cluster1.xiaohui.cn (
                        1               ; serial
                        3H              ; refresh (3 hours)
                        30M             ; retry (30 minutes)
                        2W              ; expiry (2 weeks)
                        1W )            ; minimum (1 week)
        IN      NS      ns1.cluster1.xiaohui.cn.


200.8.168.192.in-addr.arpa.     IN      PTR     ns1.cluster1.xiaohui.cn.
200.8.168.192.in-addr.arpa.     IN      PTR     content.cluster1.xiaohui.cn.
200.8.168.192.in-addr.arpa.     IN      PTR     api.cluster1.xiaohui.cn.
200.8.168.192.in-addr.arpa.     IN      PTR     api-int.cluster1.xiaohui.cn.
201.8.168.192.in-addr.arpa.     IN      PTR     bootstrap.cluster1.xiaohui.cn.
202.8.168.192.in-addr.arpa.     IN      PTR     master0.cluster1.xiaohui.cn.
203.8.168.192.in-addr.arpa.     IN      PTR     master1.cluster1.xiaohui.cn.
204.8.168.192.in-addr.arpa.     IN      PTR     master2.cluster1.xiaohui.cn.
205.8.168.192.in-addr.arpa.     IN      PTR     compute0.cluster1.xiaohui.cn.
206.8.168.192.in-addr.arpa.     IN      PTR     compute1.cluster1.xiaohui.cn. 
EOF
```

这里要注意把区域文件的owner和group设置为named

```bash
chown named:named /var/named -R
systemctl enable named --now
firewall-cmd --add-service=dns --permanent
firewall-cmd --reload
```

# 配置TFTP

在客户端从dhcp处拿到IP地址后，会根据next-server来找TFTP服务器下载内核与驱动包以及pxe配置文件，所以我们要提供TFTP服务

```bash
yum install tftp-server -y
systemctl enable tftp.service --now
systemctl enable tftp.socket --now
firewall-cmd --add-service=tftp --permanent
firewall-cmd --reload
```

# 配置HAProxy

openshift多节点通信需要一个负载均衡在前端提供服务，根据DNS的表格，我们也知道负载均衡的重要性，这里我们采用了HAProxy作为前端的负载均衡机制

```bash
yum install haproxy -y
```

在配置文件中，我们正常提供了api服务器所需的6443端口，以及新机器加入时，为机器分发配置文件的的22623端口，22623端口大家可能不熟悉，以下是它的大概描述：当新节点（主节点或工作节点）加入 OpenShift 集群时，它们需要从集群获取配置文件。这些配置文件包含了节点启动时所需的各种设置和参数。例如，网络配置、主机名、引导参数等。端口 22623 就是用于提供这些配置文件的。

我们的配置中，还为ingress提供了80和443端口用于业务，所以上面的httpd网站，我们不能占用80端口，选择了2000

```bash
cat > /etc/haproxy/haproxy.cfg <<'EOF'
global
  log         127.0.0.1 local2
  pidfile     /var/run/haproxy.pid
  maxconn     4000
  daemon
defaults
  mode                    http
  log                     global
  option                  dontlognull
  option http-server-close
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
frontend stats
  bind *:1936
  mode            http
  log             global
  maxconn 10
  stats enable
  stats hide-version
  stats refresh 30s
  stats show-node
  stats show-desc Stats for cluster
  stats auth admin:admin
  stats uri /stats
listen api-server-6443
  bind *:6443
  mode tcp
  server bootstrap bootstrap.cluster1.xiaohui.cn:6443 check inter 1s
  server master0 master0.cluster1.xiaohui.cn:6443 check inter 1s
  server master1 master1.cluster1.xiaohui.cn:6443 check inter 1s
  server master2 master2.cluster1.xiaohui.cn:6443 check inter 1s
listen machine-config-server-22623
  bind *:22623
  mode tcp
  server bootstrap bootstrap.cluster1.xiaohui.cn:22623 check inter 1s
  server master0 master0.cluster1.xiaohui.cn:22623 check inter 1s
  server master1 master1.cluster1.xiaohui.cn:22623 check inter 1s
  server master2 master2.cluster1.xiaohui.cn:22623 check inter 1s
listen ingress-router-443
  bind *:443
  mode tcp
  balance source
  server compute0 compute0.cluster1.xiaohui.cn:443 check inter 1s
  server compute1 compute1.cluster1.xiaohui.cn:443 check inter 1s
listen ingress-router-80
  bind *:80
  mode tcp
  balance source
  server compute0 compute0.cluster1.xiaohui.cn:80 check inter 1s
  server compute1 compute1.cluster1.xiaohui.cn:80 check inter 1s
EOF
```

为了让haproxy能正常工作，我们授权一下端口信息，并开通相对应的防火墙

```bash
for port in 1936 6443 22623 443 80;do semanage port -a -t http_port_t -p tcp $port;done
```

```bash
systemctl enable haproxy --now
for port in 1936 6443 22623 443 80;do firewall-cmd --add-port=$port/tcp --permanent;done
firewall-cmd --reload
```

# 配置SSH密钥

对于coreos来说，默认情况下，并没有给我们设置账户和密码的机会，默认只允许使用ssh的密钥登录，所以我们要生成公私钥，用于登录

```bash
ssh-keygen -t ed25519 -N '' -f /root/.ssh/id_ed25519
eval "$(ssh-agent -s)"
ssh-add /root/.ssh/id_ed25519
```

# 下载安装资料

这里采用了最新的稳定版，并在/content和/var/lib/tftpboot/中下载了coreos安装过程中的live、kernel、initramfs、rootfs等资料备用，以下不同版本自己选一个即可

OKD版本下载页面：https://fedoraproject.org/zh-Hans/coreos/download?stream=stable

```bash
wget -O /content/openshift-install-linux-4.11.0-0.okd-2022-08-20-022919.tar.gz https://github.com/okd-project/okd/releases/download/4.15.0-0.okd-2024-03-10-010116/openshift-install-linux-4.15.0-0.okd-2024-03-10-010116.tar.gz
wget -O /content/openshift-client-linux-4.11.0-0.okd-2022-08-20-022919.tar.gz https://github.com/okd-project/okd/releases/download/4.15.0-0.okd-2024-03-10-010116/openshift-client-linux-4.15.0-0.okd-2024-03-10-010116.tar.gz
tar Cxvf /usr/local/bin/ /content/openshift-install-linux-4.15.0-0.okd-2024-03-10-010116.tar.gz
tar Cxvf /usr/local/bin/ /content/openshift-client-linux-4.15.0-0.okd-2024-03-10-010116.tar.gz
wget -O /var/lib/tftpboot/fedora-coreos-41.20241109.3.0-live-kernel-x86_64 https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/41.20241109.3.0/x86_64/fedora-coreos-41.20241109.3.0-live-kernel-x86_64
wget -O /var/lib/tftpboot/fedora-coreos-41.20241109.3.0-live-initramfs.x86_64.img https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/41.20241109.3.0/x86_64/fedora-coreos-41.20241109.3.0-live-initramfs.x86_64.img
wget -O /content/fedora-coreos-41.20241109.3.0-live-rootfs.x86_64.img https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/41.20241109.3.0/x86_64/fedora-coreos-41.20241109.3.0-live-rootfs.x86_64.img
wget -O /content/fedora-coreos-41.20241109.3.0-live.x86_64.iso https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/41.20241109.3.0/x86_64/fedora-coreos-41.20241109.3.0-live.x86_64.iso
```

RedHat 版本直接用这里的链接即可下载到最新版

```bash
wget -O /content/openshift-install-linux.tar.gz https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/openshift-install-linux.tar.gz
wget -O /content/openshift-client-linux.tar.gz https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/openshift-client-linux.tar.gz
tar Cxvf /usr/local/bin/ /content/openshift-install-linux.tar.gz
tar Cxvf /usr/local/bin/ /content/openshift-client-linux.tar.gz
wget -O /content/rhcos-live.x86_64.iso https://mirror.openshift.com/pub/openshift-v4/x86_64/dependencies/rhcos/latest/rhcos-live.x86_64.iso
wget -O /var/lib/tftpboot/rhcos-live-kernel-x86_64 https://mirror.openshift.com/pub/openshift-v4/x86_64/dependencies/rhcos/latest/rhcos-live-kernel-x86_64
wget -O /var/lib/tftpboot/rhcos-live-initramfs.x86_64.img https://mirror.openshift.com/pub/openshift-v4/x86_64/dependencies/rhcos/latest/rhcos-live-initramfs.x86_64.img
wget -O /content/rhcos-live-rootfs.x86_64.img https://mirror.openshift.com/pub/openshift-v4/x86_64/dependencies/rhcos/latest/rhcos-live-rootfs.x86_64.img
```

# 配置私有仓库(此步骤可选)

如果你的环境可以直接连接互联网，就不需要做这一步，只有在你的机器不能连接互联网时，才需要做私有仓库步骤，将镜像都下载到私有仓库中，然后再从私有仓库来完成我们离线节点的安装

请自行生成这里所需的证书，可参考[证书签发](https://gitee.com/cnlxh/openshift/blob/master/Create-a-SANs-Certificate.md)

以下是我本次的私有仓库信息：

1. 用户名：admin
2. 密码：lixiaohui
3. 主机名：content.cluster1.xiaohui.cn

将私有仓库数据存放在/data中

```bash
mkdir /data
yum install podman jq -y
wget https://mirror.openshift.com/pub/cgw/mirror-registry/latest/mirror-registry-amd64.tar.gz
tar Cvxf /usr/local/bin mirror-registry-amd64.tar.gz
cat /root/.ssh/id_ed25519.pub >> .ssh/authorized_keys
ssh-copy-id root@content.cluster1.xiaohui.cn
mirror-registry install --initPassword lixiaohui --initUser admin \
--quayHostname content.cluster1.xiaohui.cn --quayRoot /data \
--ssh-key /root/.ssh/id_ed25519 --sslCert /etc/pki/tls/certs/xiaohui.cn.crt \
--sslKey /etc/pki/tls/private/xiaohui.cn.key --targetHostname content.cluster1.xiaohui.cn
```

quay默认工作在8443端口，别忘了开防火墙

```bash
firewall-cmd --add-port=8443/tcp --permanent
firewall-cmd --reload
```
将我们的quay私有仓库凭据初始化为base64数据
```bash
echo -n 'admin:lixiaohui' | base64 -w0
YWRtaW46bGl4aWFvaHVp
```

本地和远程拉取密钥合并

先格式化从官网下载的文件，官网的auth为了隐私，我删减了一部分，官网链接：https://console.redhat.com/openshift/install/pull-secret

```bash
cat /root/pull-secret.txt | jq . | tee /root/remote.txt
{
  "auths": {
    "cloud.openshift.com": {
      "auth": "bSkhBOEVCUOOUdVVg==",
      "email": "xiaohui_li@foxmail.com"
    },
    "quay.io": {
      "auth": "b3BlOTFOOUdVVg==",
      "email": "xiaohui_li@foxmail.com"
    },
    "registry.connect.redhat.com": {
      "auth": "fHZ1dm10eZjRQ==",
      "email": "xiaohui_li@foxmail.com"
    },
    "registry.redhat.io": {
      "auth": "fHVOTFOOUdVjRQ==",
      "email": "xiaohui_li@foxmail.com"
    }
  }
}
```

把本地的local-secret.txt也按照格式加进来，具体加完如下

```bash
[root@content ~]# cat remote.txt
{
  "auths": {
    "cloud.openshift.com": {
      "auth": "bSkhBOEVCUOOUdVVg==",
      "email": "xiaohui_li@foxmail.com"
    },
    "quay.io": {
      "auth": "b3BlOTFOOUdVVg==",
      "email": "xiaohui_li@foxmail.com"
    },
    "registry.connect.redhat.com": {
      "auth": "b3BlOTFOOUdVVg==",
      "email": "xiaohui_li@foxmail.com"
    },
    "registry.redhat.io": {
      "auth": "b3BlOTFOOUdVVg==",
      "email": "xiaohui_li@foxmail.com"
    },
    "content.cluster1.xiaohui.cn:8443": {
      "auth": "YWRtaW46bGl4aWFvaHVp",
      "email": "lixiaohui@xiaohui.cn"
    }
  }
}
```

OKD版本：

关于OCP_RELEASE版本，请参考[OpenShift OKD Release](https://quay.io/repository/openshift/okd?tab=tags)

```bash
export OCP_RELEASE='4.15.0-0.okd'
export LOCAL_REGISTRY='content.cluster1.xiaohui.cn:8443' 
export LOCAL_REPOSITORY='openshift'
export PRODUCT_REPO='openshift'
export LOCAL_SECRET_JSON='/root/remote.txt'
export RELEASE_NAME='okd'
export ARCHITECTURE='2024-03-10-010116'
```

RedHat 版本：

关于openshift版本，请参考[OpenShift](https://quay.io/openshift-release-dev/ocp-release)

```bash
export OCP_RELEASE='4.17.7'
export LOCAL_REGISTRY='content.cluster1.xiaohui.cn:8443'
export LOCAL_REPOSITORY='openshift'
export PRODUCT_REPO='openshift-release-dev'
export LOCAL_SECRET_JSON='/root/remote.txt'
export RELEASE_NAME='ocp-release'
export ARCHITECTURE='x86_64'
```

将镜像上传到私有仓库，这一步需要较久的时间，当然也取决于你的网络情况，在下载好之后，会上传到你的quay私有仓库中

```bash
oc adm -a ${LOCAL_SECRET_JSON} release mirror \
--from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE} \
--to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} \
--to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE}
```

注意结束之后的输出，我们把imageContentSources的内容放到稍后的install-config.yaml文件中

```textile
Update image:  content.cluster1.xiaohui.cn:8443/openshift:4.17.7-x86_64
Mirror prefix: content.cluster1.xiaohui.cn:8443/openshift
Mirror prefix: content.cluster1.xiaohui.cn:8443/openshift:4.17.7-x86_64

To use the new mirrored repository to install, add the following section to the install-config.yaml:

imageContentSources:
- mirrors:
  - content.cluster1.xiaohui.cn:8443/openshift
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - content.cluster1.xiaohui.cn:8443/openshift
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev


To use the new mirrored repository for upgrades, use the following to create an ImageContentSourcePolicy:

apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: example
spec:
  repositoryDigestMirrors:
  - mirrors:
    - content.cluster1.xiaohui.cn:8443/openshift
    source: quay.io/openshift-release-dev/ocp-release
  - mirrors:
    - content.cluster1.xiaohui.cn:8443/openshift
    source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
```

生成匹配版本的openshift-install文件，如果不做这个步骤，而用官方的openshift-install生成配置文件，将无法从私有仓库拉取镜像数据

```bash
oc adm -a ${LOCAL_SECRET_JSON} release extract \
--command=openshift-install "${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE}" \
--skip-verification=true --insecure=true
```

用我们生成的openshift-install替换掉原有的

```bash
mv openshift-install /usr/local/bin
```

# 拉取密钥

在openshift安装过程中，需要从官方拉取镜像，请打开下面的链接，保存好账号的拉取密钥

```textile
打开网址，登录账号，下载文件到合适的位置，例如/root
https://console.redhat.com/openshift/install/pull-secret
```

# 准备安装配置文件

| 参数            | 值          | 意义             |
| ------------- | ---------- | -------------- |
| baseDomain    | xiaohui.cn | 域名后缀           |
| metadata.name | cluster1   | 集群名称           |
| pullSecret    | XXX        | 填写真实pullSecret |
| sshKey        | XXX        | 填写真实ssh公钥      |
|               |            |                |

```bash
mkdir /openshift
```

要格外注意，这个配置文件名称必须是install-config.yaml

pullSecret 请使用自己从官方下载的，如果有私有仓库，可以加上进行加速，注意要加上仓库使用的CA证书，如果没有私有仓库，直接从官方拉取，配置文件就到sshkey结束

```bash
cat > /openshift/install-config.yaml <<'EOF'
apiVersion: v1
baseDomain: xiaohui.cn
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: cluster1
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.16.0.0/16
  machineNetwork:
  - cidr: 192.168.8.0/24
platform:
  none: {}
fips: false
pullSecret: '{"auths":{"content.cluster1.xiaohui.cn:8443":{"auth":"YWRtaW46bGl4aWFvaHVp","email":"lixiaohui@xiaohui.cn"}}}'
sshKey: 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGxL3I2R2omOzpdXslWUogEkG9jik0OGHR234SwUzyVl root@content.cluster1.xiaohui.cn'
# 请注意，如果是直接从互联网拉取镜像，就不要写入包括本行在内的下面所有内容，把本行以及下面的内容写成EOF，即可完成文件的创建
imageContentSources:
- mirrors:
  - content.cluster1.xiaohui.cn:8443/openshift
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - content.cluster1.xiaohui.cn:8443/openshift
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  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
  y9SPl5QdyAqw3gzs7Jbc+1Z+Gw==
  -----END CERTIFICATE-----
EOF
```

# 准备kubernetes清单和bootstrap配置文件

这一步将使用刚生成的匹配版本openshift-install软件来产生bootstrap、master、worker、auth等配置文件，或者从官网直拉的话，就用官方下载的openshift-install，不过需要注意的是，一旦执行了这个步骤，上面的配置文件就会消失，如果需要，请备份，在这一步配置文件生成之后的24小时内，必须完成集群安装，以免证书自动续订导致安装失败，如果打算从私有仓库拉取镜像，下方两个openshift-install命令请务必确保是用oc解压出来的版本，而不是直接从官方下载的

openshift-install命令生成配置文件和auth认证文件后，我们直接拷贝到网站目录中，用于稍后的pxe自动安装

```bash
eval "$(ssh-agent -s)"
ssh-add /root/.ssh/id_ed25519
# 执行了下面的install命令，install-config.yaml这个文件会消失，可以考虑是否备份
openshift-install create manifests --dir /openshift
openshift-install create ignition-configs --dir /openshift
cp -a /openshift/* /content
chmod a+rx /content -R
semanage fcontext -a -t httpd_sys_content_t "/content(/.*)?"
restorecon -RvF /content
systemctl restart httpd
```

# 准备PXE文件

将pxelinux.0、grubx64.efi、kernel、initramfs等资料复制到tftp供客户端拿取，以及创建pxe所需的引导配置文件和目录

OKD版本：

```bash
mkdir /var/lib/tftpboot/pxelinux.cfg
yum install syslinux -y
cp /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot/
mount /content/fedora-coreos-41.20241109.3.0-live.x86_64.iso /opt
cp /opt/isolinux/* /var/lib/tftpboot
cp /boot/efi/EFI/centos/grubx64.efi /var/lib/tftpboot/
```

BIOS 启动：

在这一步创建了6个标签，请根据实际情况，修改链接

**重要提示**

1. 注意每个APPEND部分，/dev/sda和ens192，这两部分要根据你的服务器实际硬盘和网卡名称来修改

2. 网络信息解读：

> IP: 192.168.8.201
> 网关: 192.168.8.2
> 掩码：255.255.255.0

```bash
cat > /var/lib/tftpboot/pxelinux.cfg/default <<EOF
UI vesamenu.c32
TIMEOUT 20
PROMPT 0
LABEL bootstrap
    menu label bootstrap
    KERNEL fedora-coreos-41.20241109.3.0-live-kernel-x86_64
    APPEND initrd=fedora-coreos-41.20241109.3.0-live-initramfs.x86_64.img coreos.live.rootfs_url=http://192.168.8.200:2000/fedora-coreos-41.20241109.3.0-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.8.200:2000/bootstrap.ign ip=192.168.8.201::192.168.8.2:255.255.255.0:bootstrap.cluster1.xiaohui.cn:ens192:none nameserver=192.168.8.200
LABEL master0
    menu label master0
    KERNEL fedora-coreos-41.20241109.3.0-live-kernel-x86_64
    APPEND initrd=fedora-coreos-41.20241109.3.0-live-initramfs.x86_64.img coreos.live.rootfs_url=http://192.168.8.200:2000/fedora-coreos-41.20241109.3.0-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.8.200:2000/master.ign ip=192.168.8.202::192.168.8.2:255.255.255.0:master0.cluster1.xiaohui.cn:ens192:none nameserver=192.168.8.200
LABEL master1
    menu label master1
    KERNEL fedora-coreos-41.20241109.3.0-live-kernel-x86_64
    APPEND initrd=fedora-coreos-41.20241109.3.0-live-initramfs.x86_64.img coreos.live.rootfs_url=http://192.168.8.200:2000/fedora-coreos-41.20241109.3.0-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.8.200:2000/master.ign ip=192.168.8.203::192.168.8.2:255.255.255.0:master1.cluster1.xiaohui.cn:ens192:none nameserver=192.168.8.200
LABEL master2
    menu label master2
    KERNEL fedora-coreos-41.20241109.3.0-live-kernel-x86_64
    APPEND initrd=fedora-coreos-41.20241109.3.0-live-initramfs.x86_64.img coreos.live.rootfs_url=http://192.168.8.200:2000/fedora-coreos-41.20241109.3.0-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.8.200:2000/master.ign ip=192.168.8.204::192.168.8.2:255.255.255.0:master2.cluster1.xiaohui.cn:ens192:none nameserver=192.168.8.200
LABEL compute0
    menu label compute0
    KERNEL fedora-coreos-41.20241109.3.0-live-kernel-x86_64
    APPEND initrd=fedora-coreos-41.20241109.3.0-live-initramfs.x86_64.img coreos.live.rootfs_url=http://192.168.8.200:2000/fedora-coreos-41.20241109.3.0-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.8.200:2000/worker.ign ip=192.168.8.205::192.168.8.2:255.255.255.0:compute0.cluster1.xiaohui.cn:ens192:none nameserver=192.168.8.200
LABEL compute1
    menu label compute1
    KERNEL fedora-coreos-41.20241109.3.0-live-kernel-x86_64
    APPEND initrd=fedora-coreos-41.20241109.3.0-live-initramfs.x86_64.img coreos.live.rootfs_url=http://192.168.8.200:2000/fedora-coreos-41.20241109.3.0-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.8.200:2000/worker.ign ip=192.168.8.206::192.168.8.2:255.255.255.0:compute1.cluster1.xiaohui.cn:ens192:none nameserver=192.168.8.200
EOF
```

```bash
restorecon -RvF /var/lib/tftpboot/
chmod a+rx /var/lib/tftpboot -R
```

UEFI 启动：

```bash
cat > /var/lib/tftpboot/grub.cfg <<EOF
set timeout=10
  menuentry 'bootstrap' {
  linuxefi fedora-coreos-41.20241109.3.0-live-kernel-x86_64 coreos.live.rootfs_url=http://192.168.8.200:2000/fedora-coreos-41.20241109.3.0-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.8.200:2000/bootstrap.ign ip=192.168.8.201::192.168.8.2:255.255.255.0:bootstrap.cluster1.xiaohui.cn:ens192:none nameserver=192.168.8.200
  initrdefi fedora-coreos-41.20241109.3.0-live-initramfs.x86_64.img
}
  menuentry 'master0' {
  linuxefi fedora-coreos-41.20241109.3.0-live-kernel-x86_64 coreos.live.rootfs_url=http://192.168.8.200:2000/fedora-coreos-41.20241109.3.0-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.8.200:2000/master.ign ip=192.168.8.202::192.168.8.2:255.255.255.0:master0.cluster1.xiaohui.cn:ens192:none nameserver=192.168.8.200
  initrdefi fedora-coreos-41.20241109.3.0-live-initramfs.x86_64.img
}
  menuentry 'master1' {
  linuxefi fedora-coreos-41.20241109.3.0-live-kernel-x86_64 coreos.live.rootfs_url=http://192.168.8.200:2000/fedora-coreos-41.20241109.3.0-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.8.200:2000/master.ign ip=192.168.8.203::192.168.8.2:255.255.255.0:master1.cluster1.xiaohui.cn:ens192:none nameserver=192.168.8.200
  initrdefi fedora-coreos-41.20241109.3.0-live-initramfs.x86_64.img
}
  menuentry 'master2' {
  linuxefi fedora-coreos-41.20241109.3.0-live-kernel-x86_64 coreos.live.rootfs_url=http://192.168.8.200:2000/fedora-coreos-41.20241109.3.0-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.8.200:2000/master.ign ip=192.168.8.204::192.168.8.2:255.255.255.0:master2.cluster1.xiaohui.cn:ens192:none nameserver=192.168.8.200
  initrdefi fedora-coreos-41.20241109.3.0-live-initramfs.x86_64.img
}
  menuentry 'compute0' {
  linuxefi fedora-coreos-41.20241109.3.0-live-kernel-x86_64 coreos.live.rootfs_url=http://192.168.8.200:2000/fedora-coreos-41.20241109.3.0-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.8.200:2000/worker.ign ip=192.168.8.205::192.168.8.2:255.255.255.0:compute0.cluster1.xiaohui.cn:ens192:none nameserver=192.168.8.200
  initrdefi fedora-coreos-41.20241109.3.0-live-initramfs.x86_64.img
}
  menuentry 'compute1' {
  linuxefi fedora-coreos-41.20241109.3.0-live-kernel-x86_64 coreos.live.rootfs_url=http://192.168.8.200:2000/fedora-coreos-41.20241109.3.0-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.8.200:2000/worker.ign ip=192.168.8.206::192.168.8.2:255.255.255.0:compute1.cluster1.xiaohui.cn:ens192:none nameserver=192.168.8.200
  initrdefi fedora-coreos-41.20241109.3.0-live-initramfs.x86_64.img
}

EOF
```

```bash
restorecon -RvF /var/lib/tftpboot/
chmod a+rx /var/lib/tftpboot -R
```

RedHat版本：

```bash
mkdir /var/lib/tftpboot/pxelinux.cfg
yum install syslinux -y
cp /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot/
mount /content/rhcos-live.x86_64.iso /opt
cp /opt/isolinux/* /var/lib/tftpboot
cp /boot/efi/EFI/centos/grubx64.efi /var/lib/tftpboot/
```

BIOS 启动：

```bash
cat > /var/lib/tftpboot/pxelinux.cfg/default <<EOF
UI vesamenu.c32
TIMEOUT 20
PROMPT 0
LABEL bootstrap
    menu label bootstrap
    KERNEL rhcos-live-kernel-x86_64
    APPEND initrd=rhcos-live-initramfs.x86_64.img coreos.live.rootfs_url=http://192.168.8.200:2000/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.8.200:2000/bootstrap.ign ip=192.168.8.201::192.168.8.2:255.255.255.0:bootstrap.cluster1.xiaohui.cn:ens192:none nameserver=192.168.8.200
LABEL master0
    menu label master0
    KERNEL rhcos-live-kernel-x86_64
    APPEND initrd=rhcos-live-initramfs.x86_64.img coreos.live.rootfs_url=http://192.168.8.200:2000/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.8.200:2000/master.ign ip=192.168.8.202::192.168.8.2:255.255.255.0:master0.cluster1.xiaohui.cn:ens192:none nameserver=192.168.8.200
LABEL master1
    menu label master1
    KERNEL rhcos-live-kernel-x86_64
    APPEND initrd=rhcos-live-initramfs.x86_64.img coreos.live.rootfs_url=http://192.168.8.200:2000/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.8.200:2000/master.ign ip=192.168.8.203::192.168.8.2:255.255.255.0:master1.cluster1.xiaohui.cn:ens192:none nameserver=192.168.8.200
LABEL master2
    menu label master2
    KERNEL rhcos-live-kernel-x86_64
    APPEND initrd=rhcos-live-initramfs.x86_64.img coreos.live.rootfs_url=http://192.168.8.200:2000/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.8.200:2000/master.ign ip=192.168.8.204::192.168.8.2:255.255.255.0:master2.cluster1.xiaohui.cn:ens192:none nameserver=192.168.8.200
LABEL compute0
    menu label compute0
    KERNEL rhcos-live-kernel-x86_64
    APPEND initrd=rhcos-live-initramfs.x86_64.img coreos.live.rootfs_url=http://192.168.8.200:2000/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.8.200:2000/worker.ign ip=192.168.8.205::192.168.8.2:255.255.255.0:compute0.cluster1.xiaohui.cn:ens192:none nameserver=192.168.8.200
LABEL compute1
    menu label compute1
    KERNEL rhcos-live-kernel-x86_64
    APPEND initrd=rhcos-live-initramfs.x86_64.img coreos.live.rootfs_url=http://192.168.8.200:2000/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.8.200:2000/worker.ign ip=192.168.8.206::192.168.8.2:255.255.255.0:compute1.cluster1.xiaohui.cn:ens192:none nameserver=192.168.8.200
EOF
```

```bash
restorecon -RvF /var/lib/tftpboot/
chmod a+rx /var/lib/tftpboot -R
```

UEFI 启动：

```bash
cat > /var/lib/tftpboot/grub.cfg <<EOF
set timeout=10
  menuentry 'bootstrap' {
  linuxefi rhcos-live-kernel-x86_64 coreos.live.rootfs_url=http://192.168.8.200:2000/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.8.200:2000/bootstrap.ign ip=192.168.8.201::192.168.8.2:255.255.255.0:bootstrap.cluster1.xiaohui.cn:ens192:none nameserver=192.168.8.200
  initrdefi rhcos-live-initramfs.x86_64.img
}
  menuentry 'master0' {
  linuxefi rhcos-live-kernel-x86_64 coreos.live.rootfs_url=http://192.168.8.200:2000/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.8.200:2000/master.ign ip=192.168.8.202::192.168.8.2:255.255.255.0:master0.cluster1.xiaohui.cn:ens192:none nameserver=192.168.8.200
  initrdefi rhcos-live-initramfs.x86_64.img
}
  menuentry 'master1' {
  linuxefi rhcos-live-kernel-x86_64 coreos.live.rootfs_url=http://192.168.8.200:2000/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.8.200:2000/master.ign ip=192.168.8.203::192.168.8.2:255.255.255.0:master1.cluster1.xiaohui.cn:ens192:none nameserver=192.168.8.200
  initrdefi rhcos-live-initramfs.x86_64.img
}
  menuentry 'master2' {
  linuxefi rhcos-live-kernel-x86_64 coreos.live.rootfs_url=http://192.168.8.200:2000/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.8.200:2000/master.ign ip=192.168.8.204::192.168.8.2:255.255.255.0:master2.cluster1.xiaohui.cn:ens192:none nameserver=192.168.8.200
  initrdefi rhcos-live-initramfs.x86_64.img
}
  menuentry 'compute0' {
  linuxefi rhcos-live-kernel-x86_64 coreos.live.rootfs_url=http://192.168.8.200:2000/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.8.200:2000/worker.ign ip=192.168.8.205::192.168.8.2:255.255.255.0:compute0.cluster1.xiaohui.cn:ens192:none nameserver=192.168.8.200
  initrdefi rhcos-live-initramfs.x86_64.img
}
  menuentry 'compute1' {
  linuxefi rhcos-live-kernel-x86_64 coreos.live.rootfs_url=http://192.168.8.200:2000/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.8.200:2000/bootstrap.ign ip=192.168.8.206::192.168.8.2:255.255.255.0:compute1.cluster1.xiaohui.cn:ens192:none nameserver=192.168.8.200
  initrdefi rhcos-live-initramfs.x86_64.img
}

EOF
```

```bash
chmod a+rx /var/lib/tftpboot -R
```

# 等待Master节点bootstrap完成

需要注意集群安装顺序，先从pxe安装bootstrap节点，等bootstrap节点安装结束后，执行journalctl检测bootstrap服务就绪的进度，出现下方提示后，再去pxe安装master节点，具体来说：

1. 先开机bootstrap节点，并从网卡启动，出现pxe界面后，选择bootstrap字样，等待bootstrap节点安装完成
2. 等bootstrap节点启动后，看到login登录时，从content节点执行ssh登录：`ssh core@bootstrap`，由于我们用的密钥，所以这里不需要密码即可登录
3. 登录后，用下面的命令来观察bootstrap节点的安装过程

```bash
journalctl -b -f -u release-image.service -u bootkube.service
```
4. 观察bootstrap的过程，直到滚动的问题不动了，并在下面显示以下内容时，就可以认为此时bootstrap节点的大体功能已就绪，但它的本职工作是引导我们后续的真正集群

```bash
Created "99_openshift-cluster-api_worker-user-data-secret.yaml"
```

5. 此时开启master机器，并从网卡启动，出现pxe界面后，选择master字样，比如master0，等待master0节点安装完成，出现login提示符就是安装系统完毕

6. 安装了master系统之后，不代表集群正常，此时我们用下面的命令来检测我们的集群是否bootstrap完成

这里要注意，可能会卡住，很正常的，它需要较久的时间才能完成

```bash
[root@content ~]# openshift-install --dir /openshift wait-for bootstrap-complete --log-level=info
INFO Waiting up to 20m0s (until 9:32PM) for the Kubernetes API at https://api.cluster1.xiaohui.cn:6443...
INFO API v1.30.0-2368+b62823b40c2cb1-dirty up
INFO Waiting up to 30m0s (until 9:42PM) for bootstrapping to complete...
INFO It is now safe to remove the bootstrap resources
INFO Time elapsed: 15m50s
```

看到上方的`It is now safe to remove the bootstrap resources`之后，请从haproxy文件中去掉bootstrap的条目，具体来说，就是在content上，修改haproxy配置文件，在所有的监听中，去掉bootstrap的条目，因为此时，我们的master0、master1、master2等等这些机器已经可以独当一面了

# 管理方式

如果想在content机器上管理openshift，把config文件创建好就可以了

```bash
mkdir /root/.kube
cp /openshift/auth/kubeconfig /root/.kube/config
```

# 添加命令自动补齐功能

```bash
oc completion bash > /etc/bash_completion.d/oc
kubectl completion bash > /etc/bash_completion.d/kubectl
source /etc/bash_completion.d/oc
source /etc/bash_completion.d/kubectl
```

# 确认Master节点就绪

在没有worker的情况下，可能authentication、console、ingress等几个的available状态一直不对，但其他都是正常，这个命令可能要很久才会全部就绪，请耐心等，而且过程中，会各种报错，是正常的，那是正在工作的表现，以下我的状态正常是因为我这是加好worker之后的内容

如果只看到master节点是ready是不够的，一定要等到下面的available更新状态为true

```bash
[root@content ~]# oc get nodes
NAME                            STATUS   ROLES           AGE   VERSION
master0.cluster1.xiaohui.cn     Ready    master,worker   63m   v1.30.0+4f0dd4d
master1.cluster1.xiaohui.cn     Ready    master,worker   63m   v1.30.0+4f0dd4d
master2.cluster1.xiaohui.cn     Ready    master,worker   63m   v1.30.0+4f0dd4d
```

```bash
[root@content ~]# oc get co
NAME                                       VERSION                          AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
authentication                             4.17.7   True        False         False      3m31s
baremetal                                  4.17.7   True        False         False      52m
cloud-controller-manager                   4.17.7   True        False         False      58m
cloud-credential                           4.17.7   True        False         False      68m
cluster-autoscaler                         4.17.7   True        False         False      52m
config-operator                            4.17.7   True        False         False      54m
console                                    4.17.7   True        False         False      108s
csi-snapshot-controller                    4.17.7   True        False         False      52m
dns                                        4.17.7   True        False         False      52m
etcd                                       4.17.7   True        False         False      50m
image-registry                             4.17.7   True        False         False      41m
ingress                                    4.17.7   True        False         False      44m
insights                                   4.17.7   True        False         False      47m
kube-apiserver                             4.17.7   True        False         False      38m
kube-controller-manager                    4.17.7   True        False         False      49m
kube-scheduler                             4.17.7   True        False         False      48m
kube-storage-version-migrator              4.17.7   True        False         False      53m
machine-api                                4.17.7   True        False         False      52m
machine-approver                           4.17.7   True        False         False      52m
machine-config                             4.17.7   True        False         False      11m
marketplace                                4.17.7   True        False         False      52m
monitoring                                 4.17.7   True        False         False      40m
network                                    4.17.7   True        False         False      54m
node-tuning                                4.17.7   True        False         False      48m
openshift-apiserver                        4.17.7   True        False         False      38m
openshift-controller-manager               4.17.7   True        False         False      48m
openshift-samples                          4.17.7   True        False         False      44m
operator-lifecycle-manager                 4.17.7   True        False         False      52m
operator-lifecycle-manager-catalog         4.17.7   True        False         False      52m
operator-lifecycle-manager-packageserver   4.17.7   True        False         False      45m
service-ca                                 4.17.7   True        False         False      54m
storage                                    4.17.7   True        False         False      54m
```


# 添加Worker节点

除刚才提到的几个available状态不对，其他的基本更新结束之后就正常了，此时我们可以通过pxe安装更多的worker节点，安装步骤和其他的一样，开机从网卡启动，选择对应的PXE条目即可

**对于worker节点，需要集群审批证书才可以在oc get nodes中出现，请多次通过以下方式去批准证书申请请求，审批好的标准是执行oc get nodes能看到worker节点，看不到可以多次审批**

```bash
[root@content ~]# oc get csr -o name | xargs oc adm certificate approve
certificatesigningrequest.certificates.k8s.io/csr-4d24n approved
certificatesigningrequest.certificates.k8s.io/csr-8dlnx approved
certificatesigningrequest.certificates.k8s.io/csr-bk98l approved
certificatesigningrequest.certificates.k8s.io/csr-dc7p8 approved
certificatesigningrequest.certificates.k8s.io/csr-dwmnx approved
certificatesigningrequest.certificates.k8s.io/csr-g9m6x approved
certificatesigningrequest.certificates.k8s.io/csr-hbfvr approved
certificatesigningrequest.certificates.k8s.io/csr-hccdb approved
certificatesigningrequest.certificates.k8s.io/csr-hcqcc approved
certificatesigningrequest.certificates.k8s.io/csr-kglq7 approved
certificatesigningrequest.certificates.k8s.io/csr-kzc8c approved
certificatesigningrequest.certificates.k8s.io/csr-rjr8h approved
certificatesigningrequest.certificates.k8s.io/csr-sklnx approved
certificatesigningrequest.certificates.k8s.io/csr-zkvxp approved
certificatesigningrequest.certificates.k8s.io/csr-zxggg approved
certificatesigningrequest.certificates.k8s.io/system:openshift:openshift-authenticator-znwx7 approved
certificatesigningrequest.certificates.k8s.io/system:openshift:openshift-monitoring-4vw5b approved
```

# 确认Worker状态就绪

在worker添加并多次审批证书之后，所有的available状态都会是正常的，用一下命令来确认一下集群安装进度，而且这个命令还可以给我们看到用于登录的网址和账号密码

```bash
[root@content ~]# openshift-install wait-for install-complete --dir=/openshift --log-level=debug
DEBUG OpenShift Installer 4.11.0-0.okd-2022-08-20-022919
DEBUG Built from commit 7493bb2821ccd348c11aa36f05aa060b3ab6beda
DEBUG Loading Install Config...
DEBUG   Loading SSH Key...
DEBUG   Loading Base Domain...
DEBUG     Loading Platform...
DEBUG   Loading Cluster Name...
DEBUG     Loading Base Domain...
DEBUG     Loading Platform...
DEBUG   Loading Networking...
DEBUG     Loading Platform...
DEBUG   Loading Pull Secret...
DEBUG   Loading Platform...
DEBUG Using Install Config loaded from state file
INFO Waiting up to 40m0s (until 2:46AM) for the cluster at https://api.cluster1.xiaohui.cn:6443 to initialize...
DEBUG Cluster is initialized
INFO Waiting up to 10m0s (until 2:16AM) for the openshift-console route to be created...
DEBUG Route found in openshift-console namespace: console
DEBUG OpenShift console route is admitted
INFO Install complete!
INFO To access the cluster as the system:admin user when using 'oc', run
INFO     export KUBECONFIG=/openshift/auth/kubeconfig
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.cluster1.xiaohui.cn
INFO Login to the console with user: "kubeadmin", and password: "B2JR4-yVzPQ-N3V8r-Jz48V"
INFO Time elapsed: 0s
```

添加了worker节点之后持续获取oc get co状态，如果发现上面提到的authentication、console、ingress等几个的available状态一直不对，可以通过以下方式解决

当然，这种方法是删除pod，并等他重建，它重建也需要一段时间，不要着急

```bash
oc get pod -o name -n openshift-ingress | xargs oc delete -n openshift-ingress
oc get pod -o name -n openshift-authentication | xargs oc delete -n openshift-authentication
oc get pod -o name -n openshift-console | xargs oc delete -n openshift-console
```

# 确认所有节点状态

如果没有worker节点，就再去审批一下证书

```bash
[root@content ~]# oc get nodes
NAME                            STATUS   ROLES           AGE   VERSION
compute0.cluster1.xiaohui.cn    Ready    worker          22m   v1.30.0+4f0dd4d
compute1.cluster1.xiaohui.cn    Ready    worker          23m   v1.30.0+4f0dd4d
master0.cluster1.xiaohui.cn     Ready    master,worker   68m   v1.30.0+4f0dd4d
master1.cluster1.xiaohui.cn     Ready    master,worker   68m   v1.30.0+4f0dd4d
master2.cluster1.xiaohui.cn     Ready    master,worker   68m   v1.30.0+4f0dd4d
```

# 界面展示

OKD版本：

![](https://gitee.com/cnlxh/openshift/raw/master/images/okd-install-dashbord-0.png)

![](https://gitee.com/cnlxh/openshift/raw/master/images/okd-install-dashbord-1.png)

Redhat OpenShift 版本：

![](https://gitee.com/cnlxh/openshift/raw/master/images/openshift-install-dashbord-0.png)

![1](https://gitee.com/cnlxh/openshift/raw/master/images/openshift-install-dashbord-1.png)
