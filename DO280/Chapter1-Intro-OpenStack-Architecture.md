```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 课程目标

- 描述基本的 Red Hat OpenStack Platform 架构和术语。
- 使用基本命令描述容器化服务并管理容器。
- 描述 Red Hat OpenStack 的核心组件，管理 undercloud 服务，并查看 undercloud 结构。
- 描述 Red Hat OpenStack 的核心组件，管理 overcloud 服务，并查看 overcloud 结构。

# 介绍OpenStack 基础架构

## 解读课堂环境

![](https://gitee.com/cnlxh/cl210/raw/master/images/Chapter0/cl210-classroom-topology.svg)

主课堂⽹络是 172.25.250.0/24 ，⽤作 RHOSP 安装的外部公共⽹络。学员的 workstation 系统位于此⽹络上，并且配置有浏览器和⽤于使⽤ RHOSP 客⼾端实⽤程序的命令⾏⼯具。

确认一下路由器上的网络情况

```bash
[root@bastion ~]# ip a | grep "172.25"
    inet 172.25.250.254/24 brd 172.25.250.255 scope global noprefixroute eth0
    inet 172.25.252.250/24 brd 172.25.252.255 scope global noprefixroute eth1
```

在 director 上，接⼝ eth0 附加到外部⽹络。br-ctlplane ⽹桥封装了接⼝ eth1，附加到provisioning ⽹络

```bash
[root@director ~]# ip a | grep "172.25"
    inet 172.25.250.200/24 brd 172.25.250.255 scope global noprefixroute eth0
    inet 172.25.249.200/24 brd 172.25.249.255 scope global br-ctlplane
    inet 172.25.249.202/32 scope global br-ctlplane
    inet 172.25.249.201/32 scope global br-ctlplane
```

```bash
[root@director ~]# ovs-vsctl list-br
br-ctlplane
br-int
[root@director ~]# ovs-vsctl list-ifaces br-ctlplane
eth1
phy-br-ctlplane
```

undercloud 和 overcloud 节点共享许多内部管理⽹络，比如tenant ⽹络、internal API 通信⽹络、provisioning ⽹络等

```bash
[root@controller0 ~]# ip route
default via 172.25.250.254 dev br-ex
169.254.169.254 via 172.25.249.200 dev eth0
172.23.0.0/16 dev o-hm0 proto kernel scope link src 172.23.3.42
172.24.1.0/24 dev vlan10 proto kernel scope link src 172.24.1.1
172.24.2.0/24 dev vlan20 proto kernel scope link src 172.24.2.1
172.24.3.0/24 dev vlan30 proto kernel scope link src 172.24.3.1
172.24.4.0/24 dev vlan40 proto kernel scope link src 172.24.4.1
172.24.5.0/24 dev vlan50 proto kernel scope link src 172.24.5.1
172.25.249.0/24 dev eth0 proto kernel scope link src 172.25.249.56
172.25.250.0/24 dev br-ex proto kernel scope link src 172.25.250.1
```

```bash
(undercloud) [stack@director ~]$ ip route
default via 172.25.250.254 dev eth0 proto static metric 100
172.25.249.0/24 dev br-ctlplane proto kernel scope link src 172.25.249.200
172.25.250.0/24 dev eth0 proto kernel scope link src 172.25.250.200 metric 100
```

## classroom 服务器

在课程进⾏过程中，classroom 使⽤ HTTP ⽂件服务器为具体的练习提供材料。classroom 还运⾏环境的 NTP 服务。

```bash
[root@classroom ~]# grep ^allow /etc/chrony.conf
allow 172.25/16
```

```bash
(undercloud) [stack@director ~]$ chronyc tracking
Reference ID    : AC19FEFE (classroom.example.com)
Stratum         : 9
Ref time (UTC)  : Sun Dec 31 16:42:02 2023
System time     : 0.000002085 seconds slow of NTP time
```

```bash
[root@classroom ~]# ls /var/www/html/
content  index.html  materials
[root@classroom ~]# ls /var/www/html/content/
boot  courses  ks  manifests  rhel7.0  rhel8.2  rhosp16.1  rhtops  slides  ucf
[root@classroom ~]# ls /var/www/html/materials/
00-node-info.yaml-clean  developer1-finance-rc       instackenv-onenode.json  osp-db.qcow2               rhel-updates.repo
00-node-info.yaml-setup  docs                        iptables-compute         osp-small.qcow2            rmq_trace.py
aide.conf                exports-controller0         iptables-controller0     osp-web.qcow2              scapy-2.3.2-1.noarch.rpm
ansible                  exports-controller0-clean   keypairs                 overcloud-heat-templates   server_new.py
ansible-app-resolved     fstab-compute               livemig.pp               passwd-controller0         squid-config
backup-and-restore       fstab-setup                 motd.custom              pcs_resource_status        squid-iptables-rules
ceph.repo                group-controller0           motd-script.sh           qemu.conf-clean            storage-swiftrings
cinder-rbd-params.txt    heat                        nfs-controller0          qemu.conf-setup            templates.tgz
cl210_consumer           hosts                       nfs-controller0-clean    rhel-8.2-x86_64-kvm.qcow2
cl210_producer           index-cr.html               openstack.repo           rhel-dvd-director.repo
classroom-plan.tar.gz    instackenv-compreview.json  oscli-setup.sh           rhel-dvd.repo
```

## DNS 区域

classroom 系统包括 BIND DNS 服务器，包含初始课程安装中系统的解析地址。条⽬包括底层的课堂虚拟机监控程序，以及 workstation 和 classroom 系统等

课堂环境中，地址范围 172.25.250.101 ⾄ .189 是为浮动 IP 地址保留的，尽量不要占用

本课程中也会用到本地 /etc/hosts

```bash
[root@classroom ~]# cd /var/named/
[root@classroom named]# ls
172.25.zone         data     example.com.zone         named.ca     named.localhost  slaves
172.25.zone-backup  dynamic  example.com.zone-backup  named.empty  named.loopback
```

```bash
[root@workstation ~]# cat /etc/hosts | more
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

172.25.254.254  classroom.example.com classroom
172.25.254.254  content.example.com content
172.25.254.254  materials.example.com materials
150.238.199.40  satellite-dle.ole.redhat.com satellite-dle
### rht-vm-hosts file listing the entries to be appended to /etc/hosts

#172.25.250.100 power.lab.example.com power
172.25.250.200  director.lab.example.com director
172.25.250.220  utility.lab.example.com utility
172.25.250.9    workstation.lab.example.com workstation
172.25.250.254  bastion.lab.example.com bastion
# NOTE: The following uses a 249 subnet
172.25.249.201  director-ui.lab.example.com director-ui
172.25.249.201  undercloud.example.com undercloud

172.25.250.1    controller0.overcloud.example.com controller0
172.25.250.50   keystone.overcloud.example.com keystone
172.25.250.50   overcloud.example.com overcloud
172.25.250.50   dashboard.overcloud.example.com dashboard
172.25.250.2    compute0.overcloud.example.com compute0
172.25.250.3    ceph0.overcloud.example.com ceph0
172.25.250.12   compute1.overcloud.example.com compute1
172.25.250.6    computehci0.overcloud.example.com computehci0

172.25.250.101  float101.instance.example.com float101
```

## IdM 服务器

在本课程中，红帽 IdM 服务器已配置了需要的⽤⼾和组，课程 IdM 服务器在utility 系统上运⾏。需要正确的 Kerberos 凭据才能查询 IdM 服务器。svc-ldap 帐⼾是集成⽤
⼾，由 RHOSP ⾝份服务⽤于查询其他 IdM ⽤⼾帐⼾。

此密码是RedHat123^

```bash
[root@utility ~]# kinit admin
Password for admin@LAB.EXAMPLE.NET:
[root@utility ~]# ipa user-find | grep User
  User login: admin
  User login: architect1
  User login: svc-ldap
...
```

```bash
[root@utility ~]# ipa group-find | grep Group
  Group name: admin-admins
  Group name: admins
  Group name: consulting-admins
...
```

utility 服务器也提供特殊的⽹络⽤途，在后续章节中，您将创建具有唯⼀ VLAN ID 的提供商⽹络。因此 utility 服务器配置为模拟交换机，具有四个虚拟 NIC，代表 VLAN ID 101 到 104。

```bash
[root@utility ~]# ip -br a | grep eth1
eth1             UP
eth1.102@eth1    UP             10.0.102.1/24 fe80::5054:ff:fe03:dc/64
eth1.104@eth1    UP             10.0.104.1/24 fe80::5054:ff:fe03:dc/64
eth1.101@eth1    UP             10.0.101.1/24 fe80::5054:ff:fe03:dc/64
eth1.103@eth1    UP             10.0.103.1/24 fe80::5054:ff:fe03:dc/64
```

## 电源管理

在课程提供的 power 系统上，预配置的 CLI 接⼝⽤作虚拟基板管理控制器（BMC），使⽤ IPMI 管理每个 RHOSP 虚拟机，这与管理真实裸机的⽅式类似。每个虚拟机都配置了各⾃单独的 BMC 配置，在端⼝ 623 使⽤不同的侦听地址。指定 IP 地址的 IPMI 命令会定向到 power 系统，以请求底层虚拟机监控程序在由提交的 IP 地址表⽰的 VM 上发起电源状态更改。

```bash
[root@power ~]$ netstat -uln | grep 623
udp        0      0 172.25.249.103:623      0.0.0.0:*
udp        0      0 172.25.249.101:623      0.0.0.0:*
udp        0      0 172.25.249.102:623      0.0.0.0:*
udp        0      0 172.25.249.112:623      0.0.0.0:*
udp        0      0 172.25.249.106:623      0.0.0.0:*
```

```bash
[student@power ~]$ cat /etc/bmc/vms
# List the VMs that the fake ipmi will manage. Include in the list one
# VM name per line along with the proxy IP the IPMI client (ipmitool)
# will use to manage it.
#
# EXAMPLE: VM_NAME,IP_ADDRESS
controller,172.25.249.101
compute0,172.25.249.102
ceph0,172.25.249.103
compute1,172.25.249.112
computehci0,172.25.249.106
```

# 介绍容器化服务

最新版本的OpenStack 将所有主要服务作为容器运行。 每个服务都有一个与主节点隔离的命名空间。 在部署期间，OpenStack会从 Red Hat Container Catalog 中提取并部署容器镜像。

## 容器镜像和源注册表

直接从公网拉镜像太慢了，tripleo部署可以在 undercloud 节点上的端⼝ 8787 创建本地注册表并⾸先将每个所需的服务镜像⼀次提取到本地注册表。

列出所需的镜像列表

```bash
(undercloud) [stack@director ~]$ openstack tripleo container image prepare \
> default --local-push-destination \
> --output-env-file local_image.yaml
# Generated with the following on 2023-12-31T12:33:04.865220
#
#   openstack tripleo container image prepare default --local-push-destination --output-env-file local_image.yaml
#

parameter_defaults:
  ContainerImagePrepare:
  - push_destination: true
    set:
      ceph_alertmanager_image: ose-prometheus-alertmanager
      ceph_alertmanager_namespace: registry.redhat.io/openshift4
      ceph_alertmanager_tag: 4.1
      ceph_grafana_image: rhceph-4-dashboard-rhel8
      ceph_grafana_namespace: registry.redhat.io/rhceph
      ceph_grafana_tag: 4
      ceph_image: rhceph-4-rhel8
      ceph_namespace: registry.redhat.io/rhceph
      ceph_node_exporter_image: ose-prometheus-node-exporter
      ceph_node_exporter_namespace: registry.redhat.io/openshift4
      ceph_node_exporter_tag: v4.1
      ceph_prometheus_image: ose-prometheus
      ceph_prometheus_namespace: registry.redhat.io/openshift4
      ceph_prometheus_tag: 4.1
      ceph_tag: latest
      name_prefix: openstack-
      name_suffix: ''
      namespace: registry.redhat.io/rhosp-rhel8
      neutron_driver: ovn
      rhel_containers: false
      tag: '16.1'
    tag_from_label: '{version}-{release}'
```

查看本地现有的镜像列表

```bash
[root@director ~]# podman images
REPOSITORY                                                                                                            TAG       IMAGE ID       CREATED       SIZE
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-swift-object                16.1-55   1d8c985afd81   3 years ago   695 MB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-neutron-server              16.1-53   74dfee7bc620   3 years ago   1.06 GB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-neutron-l3-agent            16.1-51   dbbad7c440bb   3 years ago   1.07 GB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-mistral-api                 16.1-51   7c6833dec0c9   3 years ago   1.1 GB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-nova-api                    16.1-55   e228c67575a8   3 years ago   1.13 GB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-neutron-openvswitch-agent   16.1-53   f0edb026c05e   3 years ago   920 MB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-zaqar-wsgi                  16.1-51   e03a9f37d88a   3 years ago   687 MB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-ironic-conductor            16.1-51   935759d55b09   3 years ago   903 MB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-nova-scheduler              16.1-53   29078ec6058c   3 years ago   1.26 GB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-keystone                    16.1-50   7deabb5b6e47   3 years ago   736 MB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-swift-proxy-server          16.1-50   e9f9f8ecd1b3   3 years ago   740 MB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-neutron-dhcp-agent          16.1-51   5d6a113bf44e   3 years ago   1.07 GB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-glance-api                  16.1-47   5c5b1930cd96   3 years ago   970 MB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-mistral-engine              16.1-52   5661f7b7d3c6   3 years ago   1.08 GB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-heat-engine                 16.1-51   af83b6596fff   3 years ago   897 MB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-swift-account               16.1-50   6759a357497b   3 years ago   695 MB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-ironic-pxe                  16.1-51   fe0c73d47517   3 years ago   764 MB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-swift-container             16.1-51   84dae609c44d   3 years ago   695 MB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-mistral-executor            16.1-62   dde9afb5cf2e   3 years ago   1.5 GB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-heat-api                    16.1-51   4d4612a8f640   3 years ago   897 MB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-ironic-neutron-agent        16.1-54   c55aff21979f   3 years ago   920 MB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-ironic-api                  16.1-51   b7d635c35a4d   3 years ago   758 MB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-nova-compute-ironic         16.1-54   6802409ed58c   3 years ago   1.9 GB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-nova-conductor              16.1-53   7a120462042b   3 years ago   1.05 GB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-mistral-event-engine        16.1-53   4f03510be342   3 years ago   1.08 GB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-ironic-inspector            16.1-55   8d5c0a04bdb2   3 years ago   661 MB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-placement-api               16.1-55   1ddaea6fdc7f   3 years ago   612 MB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-tempest                     16.1-56   c61fe83c250a   3 years ago   943 MB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-rabbitmq                    16.1-57   fc63a558787f   3 years ago   567 MB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-haproxy                     16.1-57   007cc42591fc   3 years ago   523 MB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-cron                        16.1-57   84b32ff4015f   3 years ago   390 MB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-keepalived                  16.1-56   e765f910ac00   3 years ago   404 MB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-memcached                   16.1-57   f215559b8344   3 years ago   411 MB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-iscsid                      16.1-56   79890bbdaee3   3 years ago   395 MB
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-mariadb                     16.1-59   8e3530d51950   3 years ago   738 MB
```

列出特定的字段

```bash
[root@director ~]# podman images --format="{{.Repository}}:{{.Tag}}"
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-swift-object:16.1-55
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-neutron-server:16.1-53
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-neutron-l3-agent:16.1-51
```

可用的完整字段获取方式如下：

```bash
podman images --format="{{json .}}"
```

## 容器基本命令

若要列出某个特定节点上运⾏的容器，请使⽤ podman ps：

```bash
[root@director ~]# podman ps
CONTAINER ID  IMAGE                                        COMMAND               CREATED         STATUS             PORTS  NAMES

7a3eb90360f6  openstack-nova-compute-ironic:16.1-54        kolla_start           23 months ago   Up 39 minutes ago         nova_compute
```

内容太多了的话，可以加入各种的filter和format定制返回内容，可用的filter和format参数参加一下链接:

```textile
https://docs.podman.io/en/latest/markdown/podman-ps.1.html#filter-f
https://docs.podman.io/en/latest/markdown/podman-ps.1.html#format-format
```

Exited 服务状态并不是故障。执⾏⼀次性配置的服务将会在它们完成时退出

```bash
[root@director ~]# podman ps -a --format="table {{.Names}} {{.Status}}" | grep heat
heat_api_cron                                                Up 46 minutes ago
heat_api                                                     Up 46 minutes ago
heat_engine_db_sync                                          Exited (0) 23 months ago
heat_engine                                                  Up 46 minutes ago
heat_init_log                                                Exited (0) 3 years ago
```

podman stats 命令显⽰实时容器资源使⽤情况统计数据。

```bash
[root@director ~]# podman stats nova_compute
ID             NAME           CPU %   MEM USAGE / LIMIT   MEM %   NET IO    BLOCK IO            PIDS
7a3eb90360f6   nova_compute   --      149.5MB / 7.608GB   1.96%   -- / --   25.98MB / 45.06kB   2
ID             NAME           CPU %   MEM USAGE / LIMIT   MEM %   NET IO    BLOCK IO            PIDS
7a3eb90360f6   nova_compute   --      149.5MB / 7.608GB   1.96%   -- / --   25.98MB / 45.06kB   2
ID             NAME           CPU %   MEM USAGE / LIMIT   MEM %   NET IO    BLOCK IO            PIDS
7a3eb90360f6   nova_compute   9.31%   149.5MB / 7.608GB   1.96%   -- / --   25.98MB / 45.06kB   2
ID             NAME           CPU %   MEM USAGE / LIMIT   MEM %   NET IO    BLOCK IO            PIDS
7a3eb90360f6   nova_compute   1.08%   149.5MB / 7.608GB   1.96%   -- / --   25.98MB / 45.06kB   2
ID             NAME           CPU %   MEM USAGE / LIMIT   MEM %   NET IO    BLOCK IO            PIDS
7a3eb90360f6   nova_compute   7.78%   149.5MB / 7.608GB   1.96%   -- / --   25.98MB / 45.06kB   2
```

使⽤ podman images 命令来显⽰所有可⽤镜像的存储库名称、⼤⼩、标记、镜像 ID 和创建⽇期。

```bash
[root@director ~]# podman images
REPOSITORY                                                                                                            TAG       IMAGE ID       CREATED       SIZE
director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-swift-object                16.1-55   1d8c985afd81   3 years ago   695 MB
```

podman inspect 命令以 JSON 格式显⽰容器配置。使⽤ jq 命令从 JSON 输出中过滤或提取数据。

```json
[root@director ~]# podman inspect nova_compute | more
[
    {
        "Id": "7a3eb90360f6369aef9b4d59e11f1fed9aac6ef808be9eeadcbd16c9a7128e92",
        "Created": "2022-01-21T09:13:05.398208492-05:00",
        "Path": "dumb-init",
        "Args": [
            "--single-child",
            "--",
            "kolla_start"
        ],
        "State": {
            "OciVersion": "1.0.1-dev",
            "Status": "running",
            "Running": true,
            "Paused": false,
            "Restarting": false,
            "OOMKilled": false,
            "Dead": false,
            "Pid": 2557,
            "ConmonPid": 2503,
            "ExitCode": 0,
            "Error": "",
            "StartedAt": "2023-12-31T12:15:26.262182764-05:00",
            "FinishedAt": "2022-02-25T03:28:28.773966842-05:00",
            "Healthcheck": {
                "Status": "",
                "FailingStreak": 0,
                "Log": null
            }
```

**jq技巧**

**获取第一级所有字段**

```bash
[root@director ~]# podman inspect nova_compute | jq '.[0] | keys'
[
  "AppArmorProfile",
  "Args",
  "BoundingCaps",
  "Config",
  "ConmonPidFile",
  "Created",
  "Dependencies",
  "Driver",
  "EffectiveCaps",
  "ExecIDs",
  "ExitCommand",
```

**查看其第二级参数**

```bash
[root@director ~]# podman inspect nova_compute | jq '.[0].State | keys'
[
  "ConmonPid",
  "Dead",
  "Error",
  "ExitCode",
  "FinishedAt",
  "Healthcheck",
  "OOMKilled",
  "OciVersion",
  "Paused",
  "Pid",
  "Restarting",
  "Running",
  "StartedAt",
  "Status"
]
```

使⽤ podman logs 命令来显⽰容器的控制台⽇志。

```bash
[root@director ~]# podman logs keystone | more
+ sudo -E kolla_set_configs
INFO:__main__:Loading config file at /var/lib/kolla/config_files/config.json
INFO:__main__:Validating config file
INFO:__main__:Kolla config strategy set to: COPY_ALWAYS
INFO:__main__:Copying service configuration files
INFO:__main__:Deleting /etc/keystone/fernet-keys
INFO:__main__:Creating directory /etc/keystone/fernet-keys
INFO:__main__:Copying /var/lib/kolla/config_files/src/etc/keystone/fernet-keys/0 to /etc/keystone/fernet-keys/0
INFO:__main__:Copying /var/lib/kolla/config_files/src/etc/keystone/fernet-keys/1 to /etc/keystone/fernet-keys/1
INFO:__main__:Deleting /etc/httpd/conf.d
```

请使⽤ podman exec 命令在容器中运⾏命令

```bash
[root@director ~]# podman exec -it keystone /bin/bash
()[root@director /]# ls
bin  boot  dev  etc  home  lib  lib64  lost+found  media  mnt  openstack  opt  proc  root  run  run_command  sbin  srv  sys  tmp  usr  var
()[root@director /]# exit
exit
[root@director ~]# podman exec -it keystone cat /etc/hostname
director.lab.example.com
```

## 使⽤ systemd 服务管理容器

最新版本的 RHOSP 使⽤ systemd 单元来管理服务容器⽣命周期。systemd 执⾏启动、停⽌和其他常⻅操作，像管理其他 systemd 单元和服务⼀样管理容器，使⽤ podman 命令与容器交互。

```bash
[root@director ~]# systemctl status tripleo
Display all 122 possibilities? (y or n)
tripleo_glance_api_healthcheck.service                tripleo_mysql_healthcheck.service
tripleo_glance_api_healthcheck.timer                  tripleo_mysql_healthcheck.timer
tripleo_glance_api.service                            tripleo_mysql.service
tripleo_haproxy.service                               tripleo_neutron_api_healthcheck.service
tripleo_heat_api_cron_healthcheck.service             tripleo_neutron_api_healthcheck.timer
...
```

```bash
systemctl restart tripleo_nova_compute.service
```

systemd 使⽤ systemd 定时器来监控容器健康检查，若要列出容器计时器，请使⽤ systemctl list-timers 命令。

需要注意的是，timer一般会和同名的service关联，除非你指定了Portof参数，下面我们查了一个timer的内容

```bash
[root@director ~]# systemctl cat tripleo_heat_api_healthcheck.timer
# /etc/systemd/system/tripleo_heat_api_healthcheck.timer
[Unit]
Description=heat_api container healthcheck
PartOf=tripleo_heat_api.service
[Timer]
OnActiveSec=120
OnUnitActiveSec=60
RandomizedDelaySec=45.0
[Install]
WantedBy=timers.target
```

 文件分解

1. **[Unit]** 部分：
   
   - **Description**: 描述定时器的用途。
   
   - **PartOf**: 指定这个定时器属于 `tripleo_heat_api.service`。

2. **[Timer]** 部分：
   
   - **OnActiveSec=120**: 定时器将在服务激活后 120 秒启动。
   
   - **OnUnitActiveSec=60**: 定时器将在服务的上一个活动周期结束 60 秒后再次启动。
   
   - **RandomizedDelaySec=45.0**: 每次定时器启动前增加最多 45 秒的随机延迟。

3. **[Install]** 部分：
   
   - **WantedBy=timers.target**: 指定这个定时器将在 `timers.target` 启动时启动。

你要验证的话，可以不断的去systemctl status tripleo_heat_api_healthcheck，你会看到60多秒后，服务会启动一次

## ⽇志⽂件位置

标准输出（stdout）和标准错误（stderr）整合到每个容器的⼀个⽂件中，该⽂件位于 /var/log/containers/stdouts ⽬录中

```bash
[root@director ~]# ls -1 /var/log/containers/stdouts/ | more
container-puppet-crond.log
container-puppet-glance_api.log
container-puppet-haproxy.log
container-puppet-heat_api.log
container-puppet-heat.log
container-puppet-ironic_api.log
container-puppet-ironic_inspector.log
```

## 配置⽂件位置

容器配置⽂件位于 /var/lib/config-data/puppet-generated/container_name ⽬录中

```bash
[root@director ~]# ls -1 /var/lib/config-data/puppet-generated/ | more
crond
crond.md5sum
glance_api
glance_api.md5sum
haproxy
haproxy.md5sum
heat
heat_api
heat_api.md5sum
heat.md5sum
```

容器化服务的 systemd 单元⽂件使⽤前缀 tripleo_ 进⾏命名，因为它们是由 TripleO 安装的。

```bash
[root@director ~]# ls -1 /etc/systemd/system/tripleo* | more
/etc/systemd/system/tripleo_glance_api_healthcheck.service
/etc/systemd/system/tripleo_glance_api_healthcheck.timer
/etc/systemd/system/tripleo_glance_api.service
/etc/systemd/system/tripleo_haproxy.service
/etc/systemd/system/tripleo_heat_api_cron_healthcheck.service
/etc/systemd/system/tripleo_heat_api_cron_healthcheck.timer
```

# 描述 Undercloud

## 介绍 Undercloud

Director undercloud 节点是⼀体化的红帽 OpenStack 平台（RHOSP）部署，提供⽤于安装和管理 OpenStack overcloud 的服务。部署⼯具集是 TripleO Deployment 服务。TripleO 使⽤其他现有的 OpenStack 组件（如 Heat 和 Ansible Playbook）将裸机系统置备、部署并配置为OpenStack 云节点。

undercloud 是 overcloud 的部署云，其中的⼯作负载是 overcloud 节点本⾝，例如控制器、计算和存储等节点。由于基础架构节点通常构建在物理硬件系统上，因此 undercloud 可以被称为裸机恢复云。在本课程中，undercloud 构建在虚拟机上，以便更轻松地在学习环境中部署和提供。undercloud 组件提供多种功能：

**身份服务 （Keystone）**

Identity 服务提供用户身份验证和授权，但仅限于 undercloud 的 OpenStack 服务。

**镜像服务 （Glance）**

Image 服务存储要部署到裸机节点的初始映像。 这些映像包含 Red Hat Enterprise Linux （RHEL） 操作系统、KVM 虚拟机管理程序和容器运行时。

**计算服务 （Nova）**

Compute 服务与裸机服务配合使用，通过内省可用系统来获取硬件属性，从而预置节点。 Compute 服务的调度功能会筛选可用节点，以确保所选节点满足角色要求。

**裸机服务 （Ironic）**

裸机服务管理和预置物理机。 `ironic-inspector`服务通过 PXE 引导未注册的硬件来执行内省。 undercloud 使用带外管理接口（如 IPMI）在内省期间执行电源管理。

**编排服务 （Heat）**

Orchestration 服务提供了一组 YAML 模板和节点角色，用于定义 overcloud 部署的配置和供应说明。 默认编排模板位于`/usr/share/openstack-tripleo-heat-templates` ，旨在使用环境参数文件进行自定义。

**对象服务 （Swift）**

undercloud 对象存储包含镜像、部署日志和内省结果。

**联网服务 （Neutron）**

联⽹服务为所需的 provisioning ⽹络和 external ⽹络配置接⼝。 provisioning ⽹络为裸机节点提供 DHCP 和 PXE 引导功能。external ⽹络提供公共连接。

## 查看 Undercloud 上的服务

undercloud 安装会在 stack ⽤⼾的主⽬录中创建 stackrc ⾝份环境⽂件，并⾃动从其 .bashrc⽂件获得。获得之后，此⽂件提供⽤于访问 undercloud 上服务的 admin 凭据。stackrc ⽂件中的OS_AUTH_URL 环境变量指向课堂环境中 undercloud 的⾝份服务公共端点。

**列出在 undercloud 上注册的服务**

```bash
(undercloud) [stack@director ~]$ source stackrc
(undercloud) [stack@director ~]$ openstack service list
+----------------------------------+------------------+-------------------------+
| ID                               | Name             | Type                    |
+----------------------------------+------------------+-------------------------+
| 2a08c08cc51a4f299536fc66e4b748b6 | nova             | compute                 |
| 313b7a22ef534d3fa367f50d7c9e2754 | mistral          | workflowv2              |
| 417c10b71acb4c1aa40169e251b3d16d | zaqar-websocket  | messaging-websocket     |
| 4d957e71f3284818b3a0617218f446bd | neutron          | network                 |
| 6402d082e73a459c93d5e0b70783b7e5 | placement        | placement               |
| 6f778463089446f49905f6842c25d92e | ironic-inspector | baremetal-introspection |
| 7aa774aa28c344e49aa9eb01de4900ec | zaqar            | messaging               |
| a4158e3f67a1472cb0798ebc979e4e3a | ironic           | baremetal               |
| a822dfa8c6da4695a702b46e38d0077d | heat             | orchestration           |
| ba7910222ac042c6a847a9a3c3c5074a | glance           | image                   |
| bd92462118c042a1a55967372b4b695f | keystone         | identity                |
| d9a5f802093d48958acb7f6e857f0384 | swift            | object-store            |
+----------------------------------+------------------+-------------------------+
```

列出endpoint

```bash
(undercloud) [stack@director ~]$ openstack endpoint list -c 'Service Type' -c 'Interface' -c 'URL'
+-------------------------+-----------+----------------------------------------------------+
| Service Type            | Interface | URL                                                |
+-------------------------+-----------+----------------------------------------------------+
| placement               | internal  | http://172.25.249.202:8778/placement               |
| image                   | internal  | http://172.25.249.202:9292                         |
| baremetal               | internal  | http://172.25.249.202:6385                         |
| messaging-websocket     | admin     | ws://172.25.249.202:9000                           |
| placement               | admin     | http://172.25.249.202:8778/placement               |
| identity                | admin     | http://172.25.249.202:35357                        |
| compute                 | internal  | http://172.25.249.202:8774/v2.1                    |
| identity                | public    | https://172.25.249.201:13000                       |
| baremetal-introspection | admin     | http://172.25.249.202:5050                         |
| messaging               | internal  | http://172.25.249.202:8888                         |
| baremetal               | public    | https://172.25.249.201:13385                       |
| messaging-websocket     | internal  | ws://172.25.249.202:9000                           |
| image                   | admin     | http://172.25.249.202:9292                         |
| orchestration           | internal  | http://172.25.249.202:8004/v1/%(tenant_id)s        |
| placement               | public    | https://172.25.249.201:13778/placement             |
| image                   | public    | https://172.25.249.201:13292                       |
| compute                 | admin     | http://172.25.249.202:8774/v2.1                    |
| orchestration           | public    | https://172.25.249.201:13004/v1/%(tenant_id)s      |
| object-store            | public    | https://172.25.249.201:13808/v1/AUTH_%(tenant_id)s |
| orchestration           | admin     | http://172.25.249.202:8004/v1/%(tenant_id)s        |
| baremetal-introspection | internal  | http://172.25.249.202:5050                         |
| network                 | admin     | http://172.25.249.202:9696                         |
| network                 | public    | https://172.25.249.201:13696                       |
| workflowv2              | admin     | http://172.25.249.202:8989/v2                      |
| baremetal               | admin     | http://172.25.249.202:6385                         |
| messaging-websocket     | public    | wss://172.25.249.201:9000                          |
| object-store            | admin     | http://172.25.249.202:8080                         |
| identity                | internal  | http://172.25.249.202:5000                         |
| workflowv2              | public    | https://172.25.249.201:13989/v2                    |
| messaging               | admin     | http://172.25.249.202:8888                         |
| messaging               | public    | https://172.25.249.201:13888                       |
| workflowv2              | internal  | http://172.25.249.202:8989/v2                      |
| compute                 | public    | https://172.25.249.201:13774/v2.1                  |
| network                 | internal  | http://172.25.249.202:9696                         |
| object-store            | internal  | http://172.25.249.202:8080/v1/AUTH_%(tenant_id)s   |
| baremetal-introspection | public    | https://172.25.249.201:13050                       |
+-------------------------+-----------+----------------------------------------------------+
```

列出各个服务的密码

```bash
(undercloud) [stack@director ~]$ cat undercloud-passwords.conf
[auth]
undercloud_admin_password: B7Hk2yX2zly2tKwDrVh3TGFjp
undercloud_admin_token: B88pjtnpb7ch1bCMLUJqLdGAj
undercloud_aodh_password: LKNMhUQNwhmapU74r8k8Llraw
```

## 查看 Undercloud ⽹络

通过 DHCP 和 PXE 引导⽀持，undercloud 使⽤<mark>独⽴的⾼吞吐量置备⽹络</mark>来准备和部署 overcloud 节点。部署 overcloud 后，使⽤ undercloud 在置备⽹络中管理和更新 overcloud，与内部 overcloud和外部⼯作负载流量分隔。

```bash
(undercloud) [stack@director ~]$ cat undercloud.conf | egrep -v "(^#.*|^$)"
[DEFAULT]
container_images_file = /home/stack/containers-prepare-parameter.yaml
custom_env_files = /home/stack/custom-undercloud-params.yaml
enable_telemetry = false
generate_service_certificate = false
hieradata_override = /home/stack/hieradata.yaml
local_interface = eth1
local_ip = 172.25.249.200/24
overcloud_domain_name = overcloud.example.com
undercloud_admin_host = 172.25.249.202
undercloud_debug = false
undercloud_ntp_servers = 172.25.254.254
undercloud_public_host = 172.25.249.201
undercloud_service_certificate = /etc/pki/tls/certs/undercloud.pem
[ctlplane-subnet]
cidr = 172.25.249.0/24
dhcp_end = 172.25.249.59 # PXE结束后，分配到节点的正式IP范围
dhcp_start = 172.25.249.51
gateway = 172.25.249.200
inspection_iprange = 172.25.249.150,172.25.249.180  # PXE 引导期间临时分配到已注册的节点IP
masquerade = true
```

以简短的方式列出ip信息，br-ctlplane ⽹桥是 172.25.249.0 置备⽹络。eth0 接⼝是 172.25.250.0公共⽹络。

```bash
(undercloud) [stack@director ~]$ ip -br address
lo               UNKNOWN        127.0.0.1/8 ::1/128
eth0             UP             172.25.250.200/24 fe80::1f21:2be6:7500:dfef/64
eth1             UP             fe80::5054:ff:fe00:f9c8/64
ovs-system       DOWN
br-int           DOWN
br-ctlplane      UNKNOWN        172.25.249.200/24 172.25.249.202/32 172.25.249.201/32 fe80::5054:ff:fe00:f9c8/64
```

查看br_ctlplane网络范围

```bash
(undercloud) [stack@director ~]$ openstack subnet show ctlplane-subnet
+-------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field             | Value                                                                                                                                                   |
+-------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------+
| allocation_pools  | 172.25.249.51-172.25.249.59                                                                                                                             |
| cidr              | 172.25.249.0/24                                                                                                                                         |
| created_at        | 2020-10-22T09:22:35Z                                                                                                                                    |
| description       |                                                                                                                                                         |
| dns_nameservers   |                                                                                                                                                         |
| enable_dhcp       | True                                                                                                                                                    |
| gateway_ip        | 172.25.249.200                                                                                                     
```

## 查看置备资源

在初始置备过程中，裸机恢复服务使⽤ IPMI 为托管节点供电。默认情况下，节点 PXE 会进⾏引导，请求临时 DHCP 地址、内核镜像和 ramdisk 镜像以执⾏⽹络引导。

**查看PXE镜像**

```bash
(undercloud) [stack@director ~]$ openstack image list
+--------------------------------------+------------------------+--------+
| ID                                   | Name                   | Status |
+--------------------------------------+------------------------+--------+
| da2b80ea-5ffc-400c-bc0c-82b04facad9e | overcloud-full         | active |
| 9826607c-dff5-45b0-b0c4-78c44b8665e9 | overcloud-full-initrd  | active |
| bc188e61-99c5-4d32-8c32-e1e3d467149d | overcloud-full-vmlinuz | active |
+--------------------------------------+------------------------+--------+
```

1. **overcloud-full**
   
   - **ID**: `da2b80ea-5ffc-400c-bc0c-82b04facad9e`
   
   - **Status**: `active`
   
   - 这个镜像通常包含完整的操作系统和所有必要的组件，用于部署 OpenStack。

2. **overcloud-full-initrd**
   
   - **ID**: `9826607c-dff5-45b0-b0c4-78c44b8665e9`
   
   - **Status**: `active`
   
   - 这个是 `initrd` 镜像，包含启动时所需的基本文件和驱动程序。

3. **overcloud-full-vmlinuz**
   
   - **ID**: `bc188e61-99c5-4d32-8c32-e1e3d467149d`
   
   - **Status**: `active`
   
   - 这是压缩的 Linux 内核镜像，负责启动操作系统。

这些镜像共同作用，确保 OpenStack 环境能够顺利启动和运行。

**列出注册的节点**

```bash
(undercloud) [stack@director ~]$ openstack baremetal node list -c Name -c 'Power State' -c 'Provisioning State'
+-------------+-------------+--------------------+
| Name        | Power State | Provisioning State |
+-------------+-------------+--------------------+
| controller0 | None        | active             |
| compute0    | None        | active             |
| computehci0 | None        | active             |
| compute1    | None        | active             |
| ceph0       | None        | active             |
+-------------+-------------+--------------------+
```

## Undercloud 上的电源管理

在典型的 overcloud 部署中，节点都是物理系统（例如⼑⽚服务器或机架系统），具有⽤于远程关闭电源访问和管理的⽆⼈值守⽹络管理接⼝。本课程使⽤基于软件的裸机控制器接⼝来模拟所需的电源管理功能。

Bare Metal 服务在注册过程中为各个节点加载电源管理参数，这些参数从 instackenv-initial.json 配置⽂件加载。

```bash
(undercloud) [stack@director ~]$ cat instackenv-initial.json
{
  "nodes": [
    {
      "name": "controller0",
      "arch": "x86_64",
      "cpu": "2",
      "disk": "40",
      "memory": "8192",
      "mac": [ "52:54:00:00:f9:01" ],
      "pm_addr": "172.25.249.101",
      "pm_type": "pxe_ipmitool",
      "pm_user": "admin",
      "pm_password": "password",
      "pm_port": "623",
      "capabilities": "node:controller0,boot_option:local"
    },
```

## 执⾏ IPMI 电源管理

先查询IPMI地址

```bash
(undercloud) [stack@director ~]$ cat instackenv-initial.json | jq '.nodes[] | {name: .name, pm_addr: .pm_addr, pm_user: .pm_user, pm_password: .pm_password}'
{
  "name": "controller0",
  "pm_addr": "172.25.249.101",
  "pm_user": "admin",
  "pm_password": "password"
}
{
  "name": "compute0",
  "pm_addr": "172.25.249.102",
  "pm_user": "admin",
  "pm_password": "password"
}
{
  "name": "computehci0",
  "pm_addr": "172.25.249.106",
  "pm_user": "admin",
  "pm_password": "password"
}
{
  "name": "compute1",
  "pm_addr": "172.25.249.112",
  "pm_user": "admin",
  "pm_password": "password"
}
{
  "name": "ceph0",
  "pm_addr": "172.25.249.103",
  "pm_user": "admin",
  "pm_password": "password"
}
```

各种电源操作命令

```bash
(undercloud) [stack@director ~]$ ipmitool -I lanplus -U admin -P password -H 172.25.249.101 power status
(undercloud) [stack@director ~]$ openstack baremetal node power on ceph0
(undercloud) [stack@director ~]$ openstack baremetal node power off ceph0
```

# 描述 Overcloud

overcloud 是⽣产型⾃助服务云基础架构，⽤于部署虚拟机和 Web 规模的软件。，红帽⽀持的 overcloud 由 undercloud Director 节点使⽤Deployment 服务（TripleO）构建。

overcloud 包含许多已部署的节点，每个节点都配置为使⽤⻆⾊，预定义的 RHOSP ⻆⾊定义位于 /usr/share/openstack-tripleo-heat-templates/roles ⽬录中

**查看默认的角色和已部署的角色**

```bash
(undercloud) [stack@director ~]$ ls /usr/share/openstack-tripleo-heat-templates/roles/
BlockStorage.yaml       ComputeHCI.yaml             ComputeRBDEphemeral.yaml          ControllerOpenstack.yaml            DistributedCompute.yaml  Novacontrol.yaml
CellController.yaml     ComputeInstanceHA.yaml      ComputeRealTime.yaml              ControllerSriov.yaml                HciCephAll.yaml          ObjectStorage.yaml
CephAll.yaml            ComputeLiquidio.yaml        ComputeSriovIB.yaml               ControllerStorageDashboard.yaml     HciCephFile.yaml         README.rst
CephFile.yaml           ComputeLocalEphemeral.yaml  ComputeSriovRT.yaml               ControllerStorageNfs.yaml           HciCephMon.yaml          Standalone.yaml
CephObject.yaml         ComputeOvsDpdkRT.yaml       ComputeSriov.yaml                 Controller.yaml                     HciCephObject.yaml       Telemetry.yaml
CephStorage.yaml        ComputeOvsDpdkSriovRT.yaml  Compute.yaml                      Database.yaml                       IronicConductor.yaml     UndercloudMinion.yaml
ComputeAlt.yaml         ComputeOvsDpdkSriov.yaml    ControllerAllNovaStandalone.yaml  DistributedComputeHCIScaleOut.yaml  Messaging.yaml           Undercloud.yaml
ComputeDVR.yaml         ComputeOvsDpdk.yaml         ControllerNoCeph.yaml             DistributedComputeHCI.yaml          NetworkerSriov.yaml
ComputeHCIOvsDpdk.yaml  ComputePPC64LE.yaml         ControllerNovaStandalone.yaml     DistributedComputeScaleOut.yaml     Networker.yaml
(undercloud) [stack@director ~]$ ls /usr/share/openstack-tripleo-heat-templates/roles/ | wc -l
52
(undercloud) [stack@director ~]$ grep '^- name:' ~/templates/roles_data.yaml
- name: Controller
- name: Compute
- name: CephStorage
- name: ComputeHCI
```

**控制器**
加载了所有核⼼服务的控制平⾯节点。提供服务 API，并处理数据库、消息传递和⽹络处理功能。
**计算**
标准计算节点，⽤作虚拟机监控程序，运⾏已部署的实例和堆栈。
**CephStorage**
存储后端节点，使⽤容器化的红帽 Ceph 存储服务器和多个对象存储磁盘（OSD）。
**ComputeHCI**
具有超融合基础架构的计算节点，在⼀个节点上合并了计算和 Ceph OSD 功能。

## OpenStack 核心服务

- 块存储服务（Cinder）

- 镜像服务（Glance）

- 编排服务（Heat）

- 仪表板服务（Horizon）

- ⾝份服务（Keystone）

- OpenStack 联⽹服务（Neutron）

- 计算服务（Nova）

- 消息传递服务（Oslo）

- 对象存储（Swift）

**非核心的常见服务**

- 裸机恢复服务（Ironic）

- ⽂件共享服务（Manila）

- 负载平衡服务（Octavia）

## 管理 Overcloud

先看看都有哪些overcloud节点

```bash
(undercloud) [stack@director ~]$ openstack server list
+--------------------------------------+-------------+--------+------------------------+----------------+-----------+
| ID                                   | Name        | Status | Networks               | Image          | Flavor    |
+--------------------------------------+-------------+--------+------------------------+----------------+-----------+
| 6d712d0a-cb71-4cdc-93dc-f2d69ffc520b | controller0 | ACTIVE | ctlplane=172.25.249.56 | overcloud-full | baremetal |
| b4410745-55a6-490a-baab-803ea4eb9e64 | computehci0 | ACTIVE | ctlplane=172.25.249.54 | overcloud-full | baremetal |
| b87aaecb-3726-4afc-ab81-43d6a62ca7f0 | compute1    | ACTIVE | ctlplane=172.25.249.53 | overcloud-full | baremetal |
| 0d1b6dc2-361c-4276-9ee4-dd2dd387d075 | compute0    | ACTIVE | ctlplane=172.25.249.59 | overcloud-full | baremetal |
| 3f690637-f062-4f7c-baa6-88ab794bac17 | ceph0       | ACTIVE | ctlplane=172.25.249.58 | overcloud-full | baremetal |
+--------------------------------------+-------------+--------+------------------------+----------------+-----------+
```

## Overcloud 服务⾼可⽤性

在⾼可⽤性 overcloud 部署中，overcloud 由集群中的多个控制器节点组成。RHOSP Director在每个控制器节点上安装⼀组重复的 OpenStack 组件，并将它们作为单个服务进⾏管理。红帽 OpenStack 平台使⽤ Pacemaker 作为集群资源管理器，使⽤ HAProxy 作为负载平衡集群服务，并使⽤ MariaDB Galera 作为复制的数据库服务。由 Pacemaker 管理的每个资源都设置⼀个虚拟 IP地址，OpenStack 客⼾端使⽤该地址来请求访问服务。如果分配给该 IP 地址的控制器节点出现故障，则会将 IP 地址重新分配给其他控制器，需要注意的是，**为了节约资源，我们的课堂只有一个控制节点**

使⽤ pcs status 命令来列出常规 Pacemaker 信息、虚拟 IP 地址、服务和其他 Pacemaker 信息。

```bash
[root@controller0 ~]# pcs status
Cluster name: tripleo_cluster
Cluster Summary:
  * Stack: corosync
  * Current DC: controller0 (version 2.0.3-5.el8_2.1-4b1f869f0f) - partition with quorum
  * Last updated: Mon Jan  1 12:21:49 2024
  * Last change:  Mon Jan 24 14:09:44 2022 by root via crm_resource on controller0
  * 5 nodes configured
  * 22 resource instances configured

Node List:
  * Online: [ controller0 ]
  * GuestOnline: [ galera-bundle-0@controller0 ovn-dbs-bundle-0@controller0 rabbitmq-bundle-0@controller0 redis-bundle-0@controller0 ]

Full List of Resources:
  * Container bundle: galera-bundle [cluster.common.tag/gls-dle-dev-osp16-osp16_containers-openstack-mariadb:pcmklatest]:
    * galera-bundle-0   (ocf::heartbeat:galera):        Master controller0
  * Container bundle: rabbitmq-bundle [cluster.common.tag/gls-dle-dev-osp16-osp16_containers-openstack-rabbitmq:pcmklatest]:
    * rabbitmq-bundle-0 (ocf::heartbeat:rabbitmq-cluster):      Started controller0
  * Container bundle: redis-bundle [cluster.common.tag/gls-dle-dev-osp16-osp16_containers-openstack-redis:pcmklatest]:
    * redis-bundle-0    (ocf::heartbeat:redis): Master controller0
  * ip-172.25.249.50    (ocf::heartbeat:IPaddr2):       Started controller0
  * ip-172.25.250.50    (ocf::heartbeat:IPaddr2):       Started controller0
  * ip-172.24.1.51      (ocf::heartbeat:IPaddr2):       Started controller0
  * ip-172.24.1.50      (ocf::heartbeat:IPaddr2):       Started controller0
  * ip-172.24.3.50      (ocf::heartbeat:IPaddr2):       Started controller0
  * ip-172.24.4.50      (ocf::heartbeat:IPaddr2):       Started controller0
  * Container bundle: haproxy-bundle [cluster.common.tag/gls-dle-dev-osp16-osp16_containers-openstack-haproxy:pcmklatest]:
    * haproxy-bundle-podman-0   (ocf::heartbeat:podman):        Started controller0
  * Container bundle: ovn-dbs-bundle [cluster.common.tag/gls-dle-dev-osp16-osp16_containers-openstack-ovn-northd:pcmklatest]:
    * ovn-dbs-bundle-0  (ocf::ovn:ovndb-servers):       Master controller0
  * ip-172.24.1.52      (ocf::heartbeat:IPaddr2):       Started controller0
  * Container bundle: openstack-cinder-volume [cluster.common.tag/gls-dle-dev-osp16-osp16_containers-openstack-cinder-volume:pcmklatest]:
    * openstack-cinder-volume-podman-0  (ocf::heartbeat:podman):        Started controller0
  * Container bundle: openstack-manila-share [cluster.common.tag/gls-dle-dev-osp16-osp16_containers-openstack-manila-share:pcmklatest]:
    * openstack-manila-share-podman-0   (ocf::heartbeat:podman):        Started controller0

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
```

## 访问 Overcloud 上的服务

先将环境变量切换到overcloud才能访问

```bash
(undercloud) [stack@director ~]$ source overcloudrc
(overcloud) [stack@director ~]$ env | grep OS_
OS_IMAGE_API_VERSION=2
OS_AUTH_URL=http://172.25.250.50:5000
OS_CLOUDNAME=overcloud
OS_REGION_NAME=regionOne
OS_PROJECT_NAME=admin
OS_PROJECT_DOMAIN_NAME=Default
OS_USER_DOMAIN_NAME=Default
OS_IDENTITY_API_VERSION=3
OS_AUTH_TYPE=password
OS_NO_CACHE=True
OS_COMPUTE_API_VERSION=2.latest
OS_PASSWORD=redhat
PS1=${OS_CLOUDNAME:+($OS_CLOUDNAME)} [\u@\h \W]\$
OS_USERNAME=admin
OS_VOLUME_API_VERSION=3
```

看看overcloud里部署了什么服务

```bash
(overcloud) [stack@director ~]$ openstack service list
+----------------------------------+-----------+----------------+
| ID                               | Name      | Type           |
+----------------------------------+-----------+----------------+
| 0932e03c896a4e73b905b81a0e4e8b02 | cinderv2  | volumev2       |
| 1b5a14e5ff2d4d988dcbb7d1a31aa257 | panko     | event          |
| 1e50893246094c05b0b8b5028efa65d6 | heat      | orchestration  |
| 20ee8664c18146639e207d1dd833066b | heat-cfn  | cloudformation |
| 2ed07d8f7afd4126b279bd273d029962 | gnocchi   | metric         |
| 31442049b55040d5bb52589e0fd1b31f | nova      | compute        |
| 446c6d0b77a74d05a491393c8d4cc106 | keystone  | identity       |
| 76870267b75a4d0bb70772f586730659 | neutron   | network        |
| 8413046b29f44285a772a8bbd172dbd8 | manilav2  | sharev2        |
| bb7a5e08aad345a489ce0207684ae402 | swift     | object-store   |
| c970fbfbc71a49008760c6d9a6f3ee91 | aodh      | alarming       |
| cc386f44375343d19e1fc78e8e78db35 | cinderv3  | volumev3       |
| cddaafd6d76b424b946a8fdb47a8ed12 | manila    | share          |
| d799151c504541edacd68a08324d37c0 | glance    | image          |
| e15d67325321405b854447a92bfe866f | octavia   | load-balancer  |
| f8ebce21244748c09b20df6e9c939b73 | placement | placement      |
+----------------------------------+-----------+----------------+
```

看看overcloud的网络信息

eth0有249段的置备网络，br-ex拥有外部网络250网段

```bash
[root@controller0 ~]# ip -br address
lo               UNKNOWN        127.0.0.1/8 ::1/128
eth0             UP             172.25.249.56/24 172.25.249.50/32 fe80::5054:ff:fe00:f901/64
eth1             UP             fe80::5054:ff:fe01:1/64
eth2             UP             fe80::5054:ff:fe02:fa01/64
eth3             UP             fe80::5054:ff:fe03:1/64
eth4             UP             fe80::5054:ff:fe04:1/64
ovs-system       DOWN
br-prov1         UNKNOWN        fe80::5054:ff:fe03:1/64
genev_sys_6081   UNKNOWN        fe80::90f2:c5ff:fe42:a3ca/64
o-hm0            UNKNOWN        172.23.3.42/16 fe80::f816:3eff:fecd:dfd4/64
br-int           UNKNOWN        fe80::40e:d7ff:fe34:2944/64
br-ex            UNKNOWN        172.25.250.1/24 172.25.250.50/32 fe80::5054:ff:fe02:fa01/64
br-prov2         UNKNOWN        fe80::5054:ff:fe04:1/64
vlan10           UNKNOWN        172.24.1.1/24 172.24.1.51/32 172.24.1.50/32 172.24.1.52/32 fe80::ce8:c3ff:fe62:6f05/64
vlan20           UNKNOWN        172.24.2.1/24 fe80::d45d:caff:fe1f:4b5d/64
vlan40           UNKNOWN        172.24.4.1/24 172.24.4.50/32 fe80::84eb:a7ff:fe6d:657d/64
vlan50           UNKNOWN        172.24.5.1/24 fe80::581f:bdff:fed0:428b/64
vlan30           UNKNOWN        172.24.3.1/24 172.24.3.50/32 fe80::105b:8bff:fe61:2f9d/64
br-trunk         UNKNOWN        fe80::5054:ff:fe01:1/64
```

## 查看服务容器配置

```bash
[root@controller0 ~]# podman ps --format "table {{.Names}} {{.Status}}"
Names                              Status
openstack-manila-share-podman-0    Up 2 hours ago
openstack-cinder-volume-podman-0   Up 2 hours ago
ovn-dbs-bundle-podman-0            Up 2 hours ago
haproxy-bundle-podman-0            Up 2 hours ago
rabbitmq-bundle-podman-0           Up 2 hours ago
...
```

使⽤ podman inspect 命令来检查容器化服务的容器主机配置

可以看到很多细节

```bash
[root@controller0 ~]# podman inspect keystone | jq .[].HostConfig.Binds
[
  "/etc/pki/tls/cert.pem:/etc/pki/tls/cert.pem:ro,rprivate,rbind",
  "/etc/puppet:/etc/puppet:ro,rprivate,rbind",
  "/etc/pki/ca-trust/extracted:/etc/pki/ca-trust/extracted:ro,rprivate,rbind",
  "/var/lib/config-data/puppet-generated/keystone:/var/lib/kolla/config_files/src:ro,rprivate,rbind",
  "/var/log/containers/keystone:/var/log/keystone:rw,rprivate,rbind",
  "/dev/log:/dev/log:rw,rprivate,nosuid,rbind",
  "/etc/pki/tls/certs/ca-bundle.crt:/etc/pki/tls/certs/ca-bundle.crt:ro,rprivate,rbind",
  "/etc/pki/tls/certs/ca-bundle.trust.crt:/etc/pki/tls/certs/ca-bundle.trust.crt:ro,rprivate,rbind",
  "/var/log/containers/httpd/keystone:/var/log/httpd:rw,rprivate,rbind",
  "/etc/localtime:/etc/localtime:ro,rprivate,rbind",
  "/etc/pki/ca-trust/source/anchors:/etc/pki/ca-trust/source/anchors:ro,rprivate,rbind",
  "/var/lib/kolla/config_files/keystone.json:/var/lib/kolla/config_files/config.json:ro,rprivate,rbind",
  "/etc/hosts:/etc/hosts:ro,rprivate,rbind"
]
```

## 容器化服务的配置管理

不要进入到容器中改配置文件，不让容器一旦restart，你改的就丢失了，你要去物理机上改，改完再restart容器生效

<mark>配置文件路径为：/var/lib/config-data/puppet-generated/服务/etc</mark>

除了手工vim修改外，也可以用crudini命令

```bash
[root@controller0 puppet-generated]# podman exec -it keystone crudini --get /etc/keystone/keystone.conf DEFAULT debug
False
[root@controller0 puppet-generated]# crudini --get keystone/etc/keystone/keystone.conf DEFAULT debug
False
[root@controller0 puppet-generated]# crudini --set keystone/etc/keystone/keystone.conf DEFAULT debug True
[root@controller0 puppet-generated]# crudini --get keystone/etc/keystone/keystone.conf DEFAULT debug
True
[root@controller0 puppet-generated]# podman restart keystone
8f5f7908f01d3500d412c3168da68f0eec736e6e10c535a342c0778b884ae7c5
[root@controller0 puppet-generated]# podman exec -it keystone crudini --get /etc/keystone/keystone.conf DEFAULT debug
True
```

<mark>⽇志⽂件存储在 /var/log/containers/服务中</mark>

```bash
[root@controller0 puppet-generated]# cd /var/log/containers/keystone/
[root@controller0 keystone]# ls
keystone.log  keystone.log.1
[root@controller0 keystone]# tail -n 5 keystone.log
2024-01-01 12:46:08.651 25 DEBUG keystone.server.flask.request_processing.req_logging [req-10652ad7-b1da-410f-ba9d-b88a7bb4cb36 - - - - -] SCRIPT_NAME: `` log_request_info /usr/lib/python3.6/site-packages/keystone/server/flask/request_processing/req_logging.py:28
2024-01-01 12:46:08.651 25 DEBUG keystone.server.flask.request_processing.req_logging [req-10652ad7-b1da-410f-ba9d-b88a7bb4cb36 - - - - -] PATH_INFO: `/v3` log_request_info /usr/lib/python3.6/site-packages/keystone/server/flask/request_processing/req_logging.py:29
2024-01-01 12:46:08.652 25 DEBUG keystone.server.flask.request_processing.req_logging [req-3e36b5eb-1dee-4354-96b7-d3a2d872d9db - - - - -] REQUEST_METHOD: `GET` log_request_info /usr/lib/python3.6/site-packages/keystone/server/flask/request_processing/req_logging.py:27
2024-01-01 12:46:08.652 25 DEBUG keystone.server.flask.request_processing.req_logging [req-3e36b5eb-1dee-4354-96b7-d3a2d872d9db - - - - -] SCRIPT_NAME: `` log_request_info /usr/lib/python3.6/site-packages/keystone/server/flask/request_processing/req_logging.py:28
2024-01-01 12:46:08.652 25 DEBUG keystone.server.flask.request_processing.req_logging [req-3e36b5eb-1dee-4354-96b7-d3a2d872d9db - - - - -] PATH_INFO: `/v3` log_request_info /usr/lib/python3.6/site-packages/keystone/server/flask/request_processing/req_logging.py:29
```

<mark>较为实时的内存日志位于：/var/log/containers/stdouts</mark>

这些较为实时的日志除了cat tail之外，也可以podman logs xxxx

```bash
[root@controller0 stdouts]# tail keystone.log
2024-01-01T12:43:56.191771909+00:00 stderr F ++ . /usr/local/bin/kolla_httpd_setup
2024-01-01T12:43:56.191906902+00:00 stderr F ++++ whoami
2024-01-01T12:43:56.193585090+00:00 stderr F +++ [[ root == \r\o\o\t ]]
2024-01-01T12:43:56.193585090+00:00 stderr F +++ [[ rhel =~ debian|ubuntu ]]
2024-01-01T12:43:56.193585090+00:00 stderr F +++ rm -rf '/var/run/httpd/*' '/run/httpd/*' '/tmp/httpd*'
2024-01-01T12:43:56.199485843+00:00 stdout F Running command: '/usr/sbin/httpd -DFOREGROUND'
2024-01-01T12:43:56.199495470+00:00 stderr F +++ [[ rhel = centos ]]
2024-01-01T12:43:56.199495470+00:00 stderr F ++ ARGS=-DFOREGROUND
2024-01-01T12:43:56.199495470+00:00 stderr F + echo 'Running command: '\''/usr/sbin/httpd -DFOREGROUND'\'''
2024-01-01T12:43:56.199495470+00:00 stderr F + exec /usr/sbin/httpd -DFOREGROUND
```

## 查看 Controller 节点上的 Ceph 服务

```bash
[root@controller0 stdouts]# podman ps --format "{{.Names}}" | grep ceph
ceph-mds-controller0
ceph-mon-controller0
ceph-mgr-controller0
```

```bash
[root@controller0 stdouts]# systemctl list-units | grep ceph
ceph-mds@controller0.service          loaded active     running         Ceph MDS
ceph-mgr@controller0.service          loaded active     running         Ceph Manager
ceph-mon@controller0.service          loaded active     running         Ceph Monitor
```

看看有哪些存储池

```bash
[root@controller0 stdouts]# systemctl status ceph-mon@controller.service
● ceph-mon@controller.service - Ceph Monitor
   Loaded: loaded (/etc/systemd/system/ceph-mon@.service; disabled; vendor preset: disabled)
   Active: inactive (dead)
[root@controller0 stdouts]# podman exec ceph-mon-controller0 ceph osd pool ls
vms
volumes
images
manila_data
manila_metadata
```

看看ceph集群是否健康

```bash
[root@controller0 stdouts]# podman exec ceph-mon-controller0 ceph -s
  cluster:
    id:     96157d2f-395f-4a54-8a19-b6042465100d
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum controller0 (age 2h)
    mgr: controller0(active, since 2h)
    mds: cephfs:1 {0=controller0=up:active}
    osd: 6 osds: 6 up (since 2h), 6 in (since 3y)

  task status:
    scrub status:
        mds.controller0: idle

  data:
    pools:   5 pools, 320 pgs
    objects: 2.37k objects, 18 GiB
    usage:   24 GiB used, 96 GiB / 120 GiB avail
    pgs:     320 active+clean
```

后续参加了CL260课程后，可以了解更多ceph命令
