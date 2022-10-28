| 角色                    | IP            | 主机名                        | CPU     | 内存 | 操作系统   |
| ----------------------- | ------------- | ----------------------------- | ------- | ---- | ---------- |
| NTP\|DNS\|HAProxy\|TFTP | 172.16.50.200 | content.cluster1.xiaohui.cn   | 4-8核心 | 8G   | CentOS 8.5 |
| bootstrap               | 172.16.50.201 | bootstrap.cluster1.xiaohui.cn | 4-8核心 | 16G  |            |
| master                  | 172.16.50.202 | master1.cluster1.xiaohui.cn   | 4-8核心 | 16G  |            |
| master                  | 172.16.50.203 | master1.cluster1.xiaohui.cn   | 4-8核心 | 16G  |            |
| master                  | 172.16.50.204 | master2.cluster1.xiaohui.cn   | 4-8核心 | 16G  |            |
| compute0                | 172.16.50.205 | compute0.cluster1.xiaohui.cn  | 4-8核心 | 16G  |            |
| compute1                | 172.16.50.206 | compute1.cluster1.xiaohui.cn  | 4-8核心 | 16G  |            |

```textile
作者：李晓辉

微信：Lxh_Chat

邮箱：xiaohui_li@foxmail.com
```

# 配置DHCP

配置DHCP的主要好处就是不用在每个节点启动时手工输入一大串麻烦的网络信息以及coreos的各种参数， 直接分配固定的IP地址给节点，然后配合PXE，就可以省却手工在每台节点输入的事，非常方便，还提高了正确性

1. DHCP工作在172.16.50.0/24子网中
2. 对外提供的IP为172.16.50.201-172.16.50.254
3. 对外提供的DNS为172.16.50.200
4. 对外提供的网关为172.16.50.254
5. 根据DHCP客户端提供的厂商类别识别代码来判断应该用bios还是uefi提供服务
6. 客户端拿到上述网络信息后，以TFTP协议连接的地址为172.16.50.200，去拿pxelinux.0或grubx64.efi引导文件

```bash
yum install dhcp-server -y
```

```bash
cat > /etc/dhcp/dhcpd.conf <<EOF
option space PXE;
option arch code 93 = unsigned integer 16; # RFC4578
subnet 172.16.50.0 netmask 255.255.255.0 {
  range 172.16.50.201 172.16.50.254;
  option routers 172.16.50.254;
  option domain-name-servers 172.16.50.200;
  default-lease-time 600;
  max-lease-time 7200;
class "pxeclients" {
  match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
  next-server 172.16.50.200;
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
  fixed-address 172.16.50.201;
  server-name "bootstrap.cluster1.xiaohui.cn";
}
host master0 {
  hardware ethernet 00:50:56:94:07:6d;
  fixed-address 172.16.50.202;
  server-name "master0.cluster1.xiaohui.cn";
}
host master1 {
  hardware ethernet 00:50:56:94:5c:7d;
  fixed-address 172.16.50.203;
  server-name "master1.cluster1.xiaohui.cn";
}
host master2 {
  hardware ethernet 00:50:56:94:b6:21;
  fixed-address 172.16.50.204;
  server-name "master2.cluster1.xiaohui.cn";
}
host compute0 {
  hardware ethernet 00:50:56:94:ce:83;
  fixed-address 172.16.50.205;
  server-name "compute0.cluster1.xiaohui.cn";
}
host compute1 {
  hardware ethernet 00:50:56:94:fd:5a;
  fixed-address 172.16.50.206;
  server-name "compute1.cluster1.xiaohui.cn";
} 
EOF
```

```bash
systemctl enable dhcpd --now
firewall-cmd --add-service=dhcp --permanent
firewall-cmd --reload
```

# 配置NTP

 配置NTP时间同步可以确保时间的准确性，这在集群中，是非常重要的，此处将ntp.aliyun.com作为content机器的上游时间源，并将自己作为时间源，向172.16.0.0/16网段进行授时

```bash
yum install chrony -y
sed -i '/^pool/a\pool ntp.aliyun.com iburst' /etc/chrony.conf
sed -i '/^# allow/a\allow 172.16.0.0/16' /etc/chrony.conf
systemctl enable chronyd --now
firewall-cmd --add-service=ntp --permanent
firewall-cmd --reload
```

# 配置HTTPD

在PXE过程中，需要从网络中下载rootfs以及各个节点配置文件，这里从2000端口提供内容，内容放在/content中

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

| DNS                           | 角色       | 功能                                                       |
| ----------------------------- | ---------- | ---------------------------------------------------------- |
| content.cluster1.xiaohui.cn   | material   | 提供DHCP、HTTP、NTP、HTTP、DNS、TFTP、HAProxy              |
| api.cluster1.xiaohui.cn       | master api | 提供openshift master api接口，通常指向负载均衡             |
| api-int.cluster1.xiaohui.cn   | master api | 提供api的内部通信，通常指向负载均衡                        |
| *.apps.cluster1.xiaohui.cn    | APP        | 应用程序通配符，用于后期集群内的应用暴露，通常指向负载均衡 |
| bootstrap.cluster1.xiaohui.cn | bootstrap  | 用于准备集群，集群就绪后可删除                             |
| masterX.cluster1.xiaohui.cn   | master     | openshift master服务                                       |
| computeX.cluster1.xiaohui.cn  | worker     | openshift 计算节点服务                                     |

```bash
yum install bind bind-utils -y

sed -i 's/listen-on port 53 { 127.0.0.1; }/listen-on port 53 { any; }/g' /etc/named.conf
sed -i 's/allow-query     { localhost; };/allow-query     { any; };/g' /etc/named.conf
sed -i '/recursion yes;/a\        forward first;\n        forwarders { 114.114.114.114; };' /etc/named.conf
```

```bash
cat >> /etc/named.rfc1912.zones <<EOF
zone "cluster1.xiaohui.cn" IN {
        type master;
        file "cluster1.xiaohui.cn.zone";
};

zone "50.16.172.in-addr.arpa" IN {
        type master;
        file "cluster1.xiaohui.cn.ptr";
};
EOF
```

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
ns1.cluster1.xiaohui.cn.         IN      A       172.16.50.200
content.cluster1.xiaohui.cn.     IN      A       172.16.50.200
api.cluster1.xiaohui.cn.         IN      A       172.16.50.200
api-int.cluster1.xiaohui.cn.     IN      A       172.16.50.200
*.apps.cluster1.xiaohui.cn.      IN      A       172.16.50.200
bootstrap.cluster1.xiaohui.cn.   IN      A       172.16.50.201
master0.cluster1.xiaohui.cn.     IN      A       172.16.50.202
master1.cluster1.xiaohui.cn.     IN      A       172.16.50.203
master2.cluster1.xiaohui.cn.     IN      A       172.16.50.204
compute0.cluster1.xiaohui.cn.    IN      A       172.16.50.205
compute1.cluster1.xiaohui.cn.    IN      A       172.16.50.206
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


200.50.16.172.in-addr.arpa.     IN      PTR     ns1.cluster1.xiaohui.cn.
200.50.16.172.in-addr.arpa.     IN      PTR     content.cluster1.xiaohui.cn.
200.50.16.172.in-addr.arpa.     IN      PTR     api.cluster1.xiaohui.cn.
200.50.16.172.in-addr.arpa.     IN      PTR     api-int.cluster1.xiaohui.cn.
201.50.16.172.in-addr.arpa.     IN      PTR     bootstrap.cluster1.xiaohui.cn.
202.50.16.172.in-addr.arpa.     IN      PTR     master0.cluster1.xiaohui.cn.
203.50.16.172.in-addr.arpa.     IN      PTR     master1.cluster1.xiaohui.cn.
204.50.16.172.in-addr.arpa.     IN      PTR     master2.cluster1.xiaohui.cn.
205.50.16.172.in-addr.arpa.     IN      PTR     compute0.cluster1.xiaohui.cn.
206.50.16.172.in-addr.arpa.     IN      PTR     compute1.cluster1.xiaohui.cn. 
EOF
```

```bash
chown named:named /var/named -R
systemctl enable named --now
firewall-cmd --add-service=dns --permanent
firewall-cmd --reload
```

# 配置TFTP

在客户端拿到IP地址后，会根据next-server来找TFTP，所以我们要提供TFTP服务

```bash
yum install tftp-server -y
systemctl enable tftp.service --now
systemctl enable tftp.socket --now
firewall-cmd --add-service=tftp --permanent
firewall-cmd --reload
```

# 配置HAProxy

根据DNS的表格，我们知道负载均衡的重要性，这里我们采用了HAProxy

```bash
yum install haproxy -y
```

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

```bash
for port in 1936 6443 22623 443 80;do semanage port -a -t http_port_t -p tcp $port;done
```

```bash
systemctl enable haproxy --now
for port in 1936 6443 22623 443 80;do firewall-cmd --add-port=$port/tcp --permanent;done
firewall-cmd --reload
```

# 配置SSH密钥

对于coreos来说，默认情况下，并没有给我们设置账户和密码的机会，所以我们要生成公私钥，用于登录，也可以使用私有仓库的方法

```bash
ssh-keygen -t ed25519 -N '' -f /root/.ssh/id_ed25519
eval "$(ssh-agent -s)"
ssh-add /root/.ssh/id_ed25519
```

# 下载安装资料

这里采用了最新的稳定版，并在/content和/var/lib/tftpboot/中下载了coreos安装过程中的live、kernel、initramfs、rootfs等资料备用，以下不同版本自己选一个即可

OKD版本：

```bash
wget -O /content/openshift-install-linux-4.11.0-0.okd-2022-08-20-022919.tar.gz https://github.com/okd-project/okd/releases/download/4.11.0-0.okd-2022-08-20-022919/openshift-install-linux-4.11.0-0.okd-2022-08-20-022919.tar.gz
wget -O /content/openshift-client-linux-4.11.0-0.okd-2022-08-20-022919.tar.gz https://github.com/okd-project/okd/releases/download/4.11.0-0.okd-2022-08-20-022919/openshift-client-linux-4.11.0-0.okd-2022-08-20-022919.tar.gz
tar Cxvf /usr/local/bin/ /content/openshift-install-linux-4.11.0-0.okd-2022-08-20-022919.tar.gz
tar Cxvf /usr/local/bin/ /content/openshift-client-linux-4.11.0-0.okd-2022-08-20-022919.tar.gz
wget -O /var/lib/tftpboot/fedora-coreos-36.20220906.3.2-live-kernel-x86_64 https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/36.20220906.3.2/x86_64/fedora-coreos-36.20220906.3.2-live-kernel-x86_64
wget -O /var/lib/tftpboot/fedora-coreos-36.20220906.3.2-live-initramfs.x86_64.img https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/36.20220906.3.2/x86_64/fedora-coreos-36.20220906.3.2-live-initramfs.x86_64.img
wget -O /content/fedora-coreos-36.20220906.3.2-live-rootfs.x86_64.img https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/36.20220906.3.2/x86_64/fedora-coreos-36.20220906.3.2-live-rootfs.x86_64.img
wget -O /content/fedora-coreos-36.20220906.3.2-live.x86_64.iso https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/36.20220906.3.2/x86_64/fedora-coreos-36.20220906.3.2-live.x86_64.iso
```

RedHat 版本：

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

如果你的环境可以直接连接互联网，就不需要做这一步

请自行生成这里所需的证书，可参考[证书签发](https://gitee.com/cnlxh/openshift/blob/master/Create-a-SANs-Certificate.md)

1. 用户名：admin
2. 密码：lixiaohui
3. 主机名：content.cluster1.xiaohui.cn

```bash
mkdir /data
yum install podman jq -y
wget https://developers.redhat.com/content-gateway/rest/mirror/pub/openshift-v4/clients/mirror-registry/latest/mirror-registry.tar.gz
tar Cvxf /usr/local/bin mirror-registry.tar.gz
cat /root/.ssh/id_ed25519.pub >> .ssh/authorized_keys
ssh-copy-id root@content.cluster1.xiaohui.cn
mirror-registry install --initPassword lixiaohui --initUser admin \
--quayHostname content.cluster1.xiaohui.cn --quayRoot /data \
--ssh-key /root/.ssh/id_ed25519 --sslCert /etc/pki/tls/certs/xiaohui.cn.crt \
--sslKey /etc/pki/tls/private/xiaohui.cn.key --targetHostname content.cluster1.xiaohui.cn
```

```bash
firewall-cmd --add-port=8443/tcp --permanent
firewall-cmd --reload
```

```bash
echo -n 'admin:lixiaohui' | base64 -w0
YWRtaW46bGl4aWFvaHVp
```

```bash
cat > /root/local-secret.txt << EOF
{
  "auths": {
    "content.cluster1.xiaohui.cn:8443": { 
      "auth": "YWRtaW46bGl4aWFvaHVp", 
      "email": "lixiaohui@xiaohui.cn"
    }
  }
}
EOF
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

把本地的local-secret.txt也安装格式加进来，具体加完如下

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

重新把secret合并为一行，把出现的内容全部拷贝，用于下面配置文件的pullSecret部分，官方的secret隐私原因我删除了一部分

```bash
cat /root/remote.txt | jq -c > secret
```

OKD版本：

关于OCP_RELEASE版本，请参考[OpenShift OKD Release](https://quay.io/repository/openshift/okd?tab=tags)

```bash
export OCP_RELEASE='4.11.0-0.okd'
export LOCAL_REGISTRY='content.cluster1.xiaohui.cn:8443' 
export LOCAL_REPOSITORY='openshift' 
export PRODUCT_REPO='openshift' 
export LOCAL_SECRET_JSON='/root/secret' 
export RELEASE_NAME='okd'
export ARCHITECTURE='2022-08-20-022919'
```

RedHat 版本：

关于openshift版本，请参考[OpenShift](https://quay.io/openshift-release-dev/ocp-release)

```bash
export OCP_RELEASE='4.11.5'
export LOCAL_REGISTRY='content.cluster1.xiaohui.cn:8443' 
export LOCAL_REPOSITORY='openshift'
export PRODUCT_REPO='openshift-release-dev'
export LOCAL_SECRET_JSON='/root/secret'
export RELEASE_NAME='ocp-release'
export ARCHITECTURE='x86_64'
```

将镜像上传到私有仓库

```bash
oc adm -a ${LOCAL_SECRET_JSON} release mirror \
--from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE} \
--to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} \
--to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE}
```

注意结束之后的输出，我们把imageContentSources的内容放到稍后的install-config.yaml文件中

```textile
Update image:  content.cluster1.xiaohui.cn:8443/openshift:4.11.5-x86_64
Mirror prefix: content.cluster1.xiaohui.cn:8443/openshift
Mirror prefix: content.cluster1.xiaohui.cn:8443/openshift:4.11.5-x86_64

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

生成匹配版本的openshift-install文件，如果不做这个步骤，而用官方的openshift-install生成配置文件，将无法从私有仓库拉取

```bash
oc adm -a ${LOCAL_SECRET_JSON} release extract \
--command=openshift-install "${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE}" \
--skip-verification=true --insecure=true
```

```bash
mv openshift-install /usr/local/bin
```

# 拉取密钥

在openshift安装过程中，需要从官方拉取镜像，请打开下面的链接，保存好账号的拉取密钥，推荐使用已经合并好的密钥

```textile
打开网址，登录账号，下载文件到合适的位置，例如/root
https://console.redhat.com/openshift/install/pull-secret
```

# 准备安装配置文件

| 参数          | 值         | 意义               |
| ------------- | ---------- | ------------------ |
| baseDomain    | xiaohui.cn | 域名后缀           |
| metadata.name | cluster1   | 集群名称           |
| pullSecret    | XXX        | 填写真实pullSecret |
| sshKey        | XXX        | 填写真实ssh公钥    |
|               |            |                    |

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
  networkType: OpenShiftSDN
  serviceNetwork:
  - 192.168.0.0/16
  machineNetwork:
  - cidr: 172.16.50.0/24
platform:
  none: {}
fips: false
pullSecret: '{"auths":{"content.cluster1.xiaohui.cn:8443":{"auth":"YWRtaW46bGl4aWFvaHVp","email":"lixiaohui@xiaohui.cn"}}}'
sshKey: 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGxL3I2R2omOzpdXslWUogEkG9jik0OGHR234SwUzyVl root@content.cluster1.xiaohui.cn'
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

这一步将使用刚生成的匹配版本软件来产生bootstrap、master、worker、auth等配置文件，需要注意的是，一旦执行了这个步骤，上面的配置文件就会消失，如果需要，请备份，在这一步配置文件生成之后的24小时内，必须完成集群安装，以免证书自动续订导致安装失败，如果打算从私有仓库拉取镜像，下方两个openshift-install命令请务必确保是用oc解压出来的版本，而不是直接从官方下载的

```bash
eval "$(ssh-agent -s)"
ssh-add /root/.ssh/id_ed25519
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
mount /content/fedora-coreos-36.20220906.3.2-live.x86_64.iso /opt
cp /opt/isolinux/* /var/lib/tftpboot
cp /boot/efi/EFI/centos/grubx64.efi /var/lib/tftpboot/
```

BIOS 启动：

在这一步创建了6个标签，请根据实际情况，修改链接

```bash
cat > /var/lib/tftpboot/pxelinux.cfg/default <<EOF
UI vesamenu.c32
TIMEOUT 20
PROMPT 0
LABEL bootstrap
    menu label bootstrap
    KERNEL fedora-coreos-36.20220906.3.2-live-kernel-x86_64
    APPEND initrd=fedora-coreos-36.20220906.3.2-live-initramfs.x86_64.img coreos.live.rootfs_url=http://172.16.50.200:2000/fedora-coreos-36.20220906.3.2-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.200:2000/bootstrap.ign ip=172.16.50.201::172.16.50.254:255.255.255.0:bootstrap.cluster1.xiaohui.cn:ens192:none nameserver=172.16.50.200
LABEL master0
    menu label master0
    KERNEL fedora-coreos-36.20220906.3.2-live-kernel-x86_64
    APPEND initrd=fedora-coreos-36.20220906.3.2-live-initramfs.x86_64.img coreos.live.rootfs_url=http://172.16.50.200:2000/fedora-coreos-36.20220906.3.2-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.200:2000/master.ign ip=172.16.50.202::172.16.50.254:255.255.255.0:master0.cluster1.xiaohui.cn:ens192:none nameserver=172.16.50.200
LABEL master1
    menu label master1
    KERNEL fedora-coreos-36.20220906.3.2-live-kernel-x86_64
    APPEND initrd=fedora-coreos-36.20220906.3.2-live-initramfs.x86_64.img coreos.live.rootfs_url=http://172.16.50.200:2000/fedora-coreos-36.20220906.3.2-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.200:2000/master.ign ip=172.16.50.203::172.16.50.254:255.255.255.0:master1.cluster1.xiaohui.cn:ens192:none nameserver=172.16.50.200
LABEL master2
    menu label master2
    KERNEL fedora-coreos-36.20220906.3.2-live-kernel-x86_64
    APPEND initrd=fedora-coreos-36.20220906.3.2-live-initramfs.x86_64.img coreos.live.rootfs_url=http://172.16.50.200:2000/fedora-coreos-36.20220906.3.2-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.200:2000/master.ign ip=172.16.50.204::172.16.50.254:255.255.255.0:master2.cluster1.xiaohui.cn:ens192:none nameserver=172.16.50.200
LABEL compute0
    menu label compute0
    KERNEL fedora-coreos-36.20220906.3.2-live-kernel-x86_64
    APPEND initrd=fedora-coreos-36.20220906.3.2-live-initramfs.x86_64.img coreos.live.rootfs_url=http://172.16.50.200:2000/fedora-coreos-36.20220906.3.2-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.200:2000/worker.ign ip=172.16.50.205::172.16.50.254:255.255.255.0:compute0.cluster1.xiaohui.cn:ens192:none nameserver=172.16.50.200
LABEL compute1
    menu label compute1
    KERNEL fedora-coreos-36.20220906.3.2-live-kernel-x86_64
    APPEND initrd=fedora-coreos-36.20220906.3.2-live-initramfs.x86_64.img coreos.live.rootfs_url=http://172.16.50.200:2000/fedora-coreos-36.20220906.3.2-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.200:2000/worker.ign ip=172.16.50.206::172.16.50.254:255.255.255.0:compute1.cluster1.xiaohui.cn:ens192:none nameserver=172.16.50.200
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
  linuxefi fedora-coreos-36.20220906.3.2-live-kernel-x86_64 coreos.live.rootfs_url=http://172.16.50.200:2000/fedora-coreos-36.20220906.3.2-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.200:2000/bootstrap.ign ip=172.16.50.201::172.16.50.254:255.255.255.0:bootstrap.cluster1.xiaohui.cn:ens192:none nameserver=172.16.50.200
  initrdefi fedora-coreos-36.20220906.3.2-live-initramfs.x86_64.img
}
  menuentry 'master0' {
  linuxefi fedora-coreos-36.20220906.3.2-live-kernel-x86_64 coreos.live.rootfs_url=http://172.16.50.200:2000/fedora-coreos-36.20220906.3.2-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.200:2000/master.ign ip=172.16.50.202::172.16.50.254:255.255.255.0:master0.cluster1.xiaohui.cn:ens192:none nameserver=172.16.50.200
  initrdefi fedora-coreos-36.20220906.3.2-live-initramfs.x86_64.img
}
  menuentry 'master1' {
  linuxefi fedora-coreos-36.20220906.3.2-live-kernel-x86_64 coreos.live.rootfs_url=http://172.16.50.200:2000/fedora-coreos-36.20220906.3.2-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.200:2000/master.ign ip=172.16.50.203::172.16.50.254:255.255.255.0:master1.cluster1.xiaohui.cn:ens192:none nameserver=172.16.50.200
  initrdefi fedora-coreos-36.20220906.3.2-live-initramfs.x86_64.img
}
  menuentry 'master2' {
  linuxefi fedora-coreos-36.20220906.3.2-live-kernel-x86_64 coreos.live.rootfs_url=http://172.16.50.200:2000/fedora-coreos-36.20220906.3.2-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.200:2000/master.ign ip=172.16.50.204::172.16.50.254:255.255.255.0:master2.cluster1.xiaohui.cn:ens192:none nameserver=172.16.50.200
  initrdefi fedora-coreos-36.20220906.3.2-live-initramfs.x86_64.img
}
  menuentry 'compute0' {
  linuxefi fedora-coreos-36.20220906.3.2-live-kernel-x86_64 coreos.live.rootfs_url=http://172.16.50.200:2000/fedora-coreos-36.20220906.3.2-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.200:2000/worker.ign ip=172.16.50.205::172.16.50.254:255.255.255.0:compute0.cluster1.xiaohui.cn:ens192:none nameserver=172.16.50.200
  initrdefi fedora-coreos-36.20220906.3.2-live-initramfs.x86_64.img
}
  menuentry 'compute1' {
  linuxefi fedora-coreos-36.20220906.3.2-live-kernel-x86_64 coreos.live.rootfs_url=http://172.16.50.200:2000/fedora-coreos-36.20220906.3.2-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.200:2000/worker.ign ip=172.16.50.206::172.16.50.254:255.255.255.0:compute1.cluster1.xiaohui.cn:ens192:none nameserver=172.16.50.200
  initrdefi fedora-coreos-36.20220906.3.2-live-initramfs.x86_64.img
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
    APPEND initrd=rhcos-live-initramfs.x86_64.img coreos.live.rootfs_url=http://172.16.50.200:2000/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.200:2000/bootstrap.ign ip=172.16.50.201::172.16.50.254:255.255.255.0:bootstrap.cluster1.xiaohui.cn:ens192:none nameserver=172.16.50.200
LABEL master0
    menu label master0
    KERNEL rhcos-live-kernel-x86_64
    APPEND initrd=rhcos-live-initramfs.x86_64.img coreos.live.rootfs_url=http://172.16.50.200:2000/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.200:2000/master.ign ip=172.16.50.202::172.16.50.254:255.255.255.0:master0.cluster1.xiaohui.cn:ens192:none nameserver=172.16.50.200
LABEL master1
    menu label master1
    KERNEL rhcos-live-kernel-x86_64
    APPEND initrd=rhcos-live-initramfs.x86_64.img coreos.live.rootfs_url=http://172.16.50.200:2000/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.200:2000/master.ign ip=172.16.50.203::172.16.50.254:255.255.255.0:master1.cluster1.xiaohui.cn:ens192:none nameserver=172.16.50.200
LABEL master2
    menu label master2
    KERNEL rhcos-live-kernel-x86_64
    APPEND initrd=rhcos-live-initramfs.x86_64.img coreos.live.rootfs_url=http://172.16.50.200:2000/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.200:2000/master.ign ip=172.16.50.204::172.16.50.254:255.255.255.0:master2.cluster1.xiaohui.cn:ens192:none nameserver=172.16.50.200
LABEL compute0
    menu label compute0
    KERNEL rhcos-live-kernel-x86_64
    APPEND initrd=rhcos-live-initramfs.x86_64.img coreos.live.rootfs_url=http://172.16.50.200:2000/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.200:2000/worker.ign ip=172.16.50.205::172.16.50.254:255.255.255.0:compute0.cluster1.xiaohui.cn:ens192:none nameserver=172.16.50.200
LABEL compute1
    menu label compute1
    KERNEL rhcos-live-kernel-x86_64
    APPEND initrd=rhcos-live-initramfs.x86_64.img coreos.live.rootfs_url=http://172.16.50.200:2000/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.200:2000/worker.ign ip=172.16.50.206::172.16.50.254:255.255.255.0:compute1.cluster1.xiaohui.cn:ens192:none nameserver=172.16.50.200
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
  linuxefi rhcos-live-kernel-x86_64 coreos.live.rootfs_url=http://172.16.50.200:2000/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.200:2000/bootstrap.ign ip=172.16.50.201::172.16.50.254:255.255.255.0:bootstrap.cluster1.xiaohui.cn:ens192:none nameserver=172.16.50.200
  initrdefi rhcos-live-initramfs.x86_64.img
}
  menuentry 'master0' {
  linuxefi rhcos-live-kernel-x86_64 coreos.live.rootfs_url=http://172.16.50.200:2000/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.200:2000/master.ign ip=172.16.50.202::172.16.50.254:255.255.255.0:master0.cluster1.xiaohui.cn:ens192:none nameserver=172.16.50.200
  initrdefi rhcos-live-initramfs.x86_64.img
}
  menuentry 'master1' {
  linuxefi rhcos-live-kernel-x86_64 coreos.live.rootfs_url=http://172.16.50.200:2000/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.200:2000/master.ign ip=172.16.50.203::172.16.50.254:255.255.255.0:master1.cluster1.xiaohui.cn:ens192:none nameserver=172.16.50.200
  initrdefi rhcos-live-initramfs.x86_64.img
}
  menuentry 'master2' {
  linuxefi rhcos-live-kernel-x86_64 coreos.live.rootfs_url=http://172.16.50.200:2000/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.200:2000/master.ign ip=172.16.50.204::172.16.50.254:255.255.255.0:master2.cluster1.xiaohui.cn:ens192:none nameserver=172.16.50.200
  initrdefi rhcos-live-initramfs.x86_64.img
}
  menuentry 'compute0' {
  linuxefi rhcos-live-kernel-x86_64 coreos.live.rootfs_url=http://172.16.50.200:2000/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.200:2000/worker.ign ip=172.16.50.205::172.16.50.254:255.255.255.0:compute0.cluster1.xiaohui.cn:ens192:none nameserver=172.16.50.200
  initrdefi rhcos-live-initramfs.x86_64.img
}
  menuentry 'compute1' {
  linuxefi rhcos-live-kernel-x86_64 coreos.live.rootfs_url=http://172.16.50.200:2000/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.200:2000/bootstrap.ign ip=172.16.50.206::172.16.50.254:255.255.255.0:compute1.cluster1.xiaohui.cn:ens192:none nameserver=172.16.50.200
  initrdefi rhcos-live-initramfs.x86_64.img
}

EOF
```

```bash
chmod a+rx /var/lib/tftpboot -R
```

# 等待Master节点bootstrap完成

需要注意集群安装顺序，先从pxe安装bootstrap节点，等bootstrap节点安装结束后，执行journalctl检测bootstrap服务就绪的进度，出现下方提示后，再去pxe安装master节点

```bash
ssh core@bootstrap
journalctl -b -f -u release-image.service -u bootkube.service
```

```bash
Created "99_openshift-cluster-api_worker-user-data-secret.yaml"
```

```bash
[root@content ~]# openshift-install --dir /openshift wait-for bootstrap-complete --log-level=info
INFO Waiting up to 20m0s (until 9:32PM) for the Kubernetes API at https://api.cluster1.xiaohui.cn:6443...
INFO API v1.24.0-2368+b62823b40c2cb1-dirty up
INFO Waiting up to 30m0s (until 9:42PM) for bootstrapping to complete...
INFO It is now safe to remove the bootstrap resources
INFO Time elapsed: 15m50s
```

看到上方的complete之后，或者从journalctl中出现下方提示后，请从haproxy文件中去掉bootstrap的条目

```bash
Sep 23 14:51:25 bootstrap.cluster1.xiaohui.cn bootkube.sh[5281]: bootkube.service complete
Sep 23 14:51:25 bootstrap.cluster1.xiaohui.cn systemd[1]: bootkube.service: Deactivated successfully.
Sep 23 14:51:25 bootstrap.cluster1.xiaohui.cn systemd[1]: bootkube.service: Consumed 24.785s CPU time.
```

需要注意的是，如果多次用content重装openshfit，从第二次安装开始，就无法再用ssh的方法去连接bootstrap等openshift节点，会报告以下错误

```bash
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
```

这主要是因为我们已经在家目录保存了对应主机名或IP的密钥，只需要从家目录中删除旧的条目或者删除保存条目的文件就可以了，以下为root家目录删除文件操作，如果不想删除，只需要打开这个文件，删除对应的条目，删除的副作用是连接已保存的机器时会需要我们输入yes/no来确认是否连接，这主要是SSH配置文件默认启用了主机密钥检查，但是我们的记录不匹配导致的

```bash
rm -rf /root/.ssh/known_hosts
```

# 确认Master节点就绪

在没有worker的情况下，可能authentication、console、ingress等几个的available状态一直不对，但其他都是正常，这个命令可能要很久才会全部就绪，请耐心等

```bash
[root@content ~]# oc get co
NAME                                       VERSION                          AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
authentication                             4.11.0-0.okd-2022-08-20-022919   True        False         False      3m31s
baremetal                                  4.11.0-0.okd-2022-08-20-022919   True        False         False      52m
cloud-controller-manager                   4.11.0-0.okd-2022-08-20-022919   True        False         False      58m
cloud-credential                           4.11.0-0.okd-2022-08-20-022919   True        False         False      68m
cluster-autoscaler                         4.11.0-0.okd-2022-08-20-022919   True        False         False      52m
config-operator                            4.11.0-0.okd-2022-08-20-022919   True        False         False      54m
console                                    4.11.0-0.okd-2022-08-20-022919   True        False         False      108s
csi-snapshot-controller                    4.11.0-0.okd-2022-08-20-022919   True        False         False      52m
dns                                        4.11.0-0.okd-2022-08-20-022919   True        False         False      52m
etcd                                       4.11.0-0.okd-2022-08-20-022919   True        False         False      50m
image-registry                             4.11.0-0.okd-2022-08-20-022919   True        False         False      41m
ingress                                    4.11.0-0.okd-2022-08-20-022919   True        False         False      44m
insights                                   4.11.0-0.okd-2022-08-20-022919   True        False         False      47m
kube-apiserver                             4.11.0-0.okd-2022-08-20-022919   True        False         False      38m
kube-controller-manager                    4.11.0-0.okd-2022-08-20-022919   True        False         False      49m
kube-scheduler                             4.11.0-0.okd-2022-08-20-022919   True        False         False      48m
kube-storage-version-migrator              4.11.0-0.okd-2022-08-20-022919   True        False         False      53m
machine-api                                4.11.0-0.okd-2022-08-20-022919   True        False         False      52m
machine-approver                           4.11.0-0.okd-2022-08-20-022919   True        False         False      52m
machine-config                             4.11.0-0.okd-2022-08-20-022919   True        False         False      11m
marketplace                                4.11.0-0.okd-2022-08-20-022919   True        False         False      52m
monitoring                                 4.11.0-0.okd-2022-08-20-022919   True        False         False      40m
network                                    4.11.0-0.okd-2022-08-20-022919   True        False         False      54m
node-tuning                                4.11.0-0.okd-2022-08-20-022919   True        False         False      48m
openshift-apiserver                        4.11.0-0.okd-2022-08-20-022919   True        False         False      38m
openshift-controller-manager               4.11.0-0.okd-2022-08-20-022919   True        False         False      48m
openshift-samples                          4.11.0-0.okd-2022-08-20-022919   True        False         False      44m
operator-lifecycle-manager                 4.11.0-0.okd-2022-08-20-022919   True        False         False      52m
operator-lifecycle-manager-catalog         4.11.0-0.okd-2022-08-20-022919   True        False         False      52m
operator-lifecycle-manager-packageserver   4.11.0-0.okd-2022-08-20-022919   True        False         False      45m
service-ca                                 4.11.0-0.okd-2022-08-20-022919   True        False         False      54m
storage                                    4.11.0-0.okd-2022-08-20-022919   True        False         False      54m
```

如果只看到master节点是ready是不够的，一定要等到上面的available更新状态

```bash
[root@content ~]# oc get nodes
NAME                            STATUS   ROLES           AGE   VERSION
master0.cluster1.xiaohui.cn     Ready    master,worker   63m   v1.24.0+4f0dd4d
master1.cluster1.xiaohui.cn     Ready    master,worker   63m   v1.24.0+4f0dd4d
master2.cluster1.xiaohui.cn     Ready    master,worker   63m   v1.24.0+4f0dd4d
```

# 添加Worker节点

除刚才提到的几个available状态不对，其他的基本更新结束之后，通过pxe安装更多的worker节点

对于worker节点，需要集群审批证书才可以在oc get nodes中出现，请多次通过以下方式去批准证书申请请求，审批好的标准是执行oc get csr能看到worker节点，并看到状态是approved

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

在worker添加并多次审批证书之后，所有的available状态都会是正常的了之后，监控一下集群安装进度，我们可以看到用于登录的网址和账号密码

```bnf
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

```bash
oc get pod -o name -n openshift-ingress | xargs oc delete -n openshift-ingress
oc get pod -o name -n openshift-authentication | xargs oc delete -n openshift-authentication
oc get pod -o name -n openshift-console | xargs oc delete -n openshift-console
```

# 管理方式

如果想在content机器上管理openshift，把config文件创建好就可以了

```bash
mkdir /root/.kube
cp /openshift/auth/kubeconfig /root/.kube/config
```

```bash
[root@content ~]# oc get nodes
NAME                            STATUS   ROLES           AGE   VERSION
compute0.cluster1.xiaohui.cn    Ready    worker          22m   v1.24.0+4f0dd4d
compute1.cluster1.xiaohui.cn    Ready    worker          23m   v1.24.0+4f0dd4d
master0.cluster1.xiaohui.cn     Ready    master,worker   68m   v1.24.0+4f0dd4d
master1.cluster1.xiaohui.cn     Ready    master,worker   68m   v1.24.0+4f0dd4d
master2.cluster1.xiaohui.cn     Ready    master,worker   68m   v1.24.0+4f0dd4d
```

```bash
[root@content ~]# oc get routes.route.openshift.io -n openshift-console
NAME        HOST/PORT                                              PATH   SERVICES    PORT    TERMINATION          WILDCARD
console     console-openshift-console.apps.cluster1.xiaohui.cn            console     https   reencrypt/Redirect   None
downloads   downloads-openshift-console.apps.cluster1.xiaohui.cn          downloads   http    edge/Redirect        None
```

# 添加命令自动补齐功能

由于这里用的是追加，请勿重复执行

```bash
oc completion bash > /etc/bash_completion.d/oc
source /etc/bash_completion.d/oc
```

# 界面展示

OKD版本：

![](https://gitee.com/cnlxh/openshift/raw/master/images/okd-install-dashbord-0.png)

![](https://gitee.com/cnlxh/openshift/raw/master/images/okd-install-dashbord-1.png)

Redhat OpenShift 版本：

![](https://gitee.com/cnlxh/openshift/raw/master/images/openshift-install-dashbord-0.png)

![1](https://gitee.com/cnlxh/openshift/raw/master/images/openshift-install-dashbord-1.png)
