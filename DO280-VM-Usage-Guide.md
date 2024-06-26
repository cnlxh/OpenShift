```textile
作者：李晓辉

微信联系：lxh_chat

联系邮箱: 939958092@qq.com
```

# DO280 虚拟机使用指南

**本次课程涉及到虚拟机是我根据教材自行独立研发的，如果有任何bug还请及时联系上方的微信或邮件给我，谢谢**

# 物理机硬件要求

|CPU|内存|硬盘|
|:-:|:-:|:-:|
|至少 8 核心，推荐 16 核心|至少 32G，推荐 64G|至少200G空闲的SSD磁盘|

# 虚拟机基本信息

**请将VMware workstation的VMnet网卡的网段设置为192.168.8.0/24网段后再开机，方法如下**

打开VMware软件-->>编辑-->>虚拟网络编辑器-->>点击VMnet8(NAT模式)-->>将子网IP设置为:192.168.8.0/24，掩码设置为255.255.255.0

**请确保workstation已经正常开机看到桌面后，再开master01**

**请勿修改虚拟机中的任何网络信息，日常练习执行命令都在workstation这一台机器上**

|开机顺序|主机名|CPU|内存|硬盘|VMware网卡类型|IP|角色|备注|
|:-:|:-:|:-:|:-:|:-:|:-|:-|:-|:-|
|1|workstation|8核心|16G内存，推荐20G|500G|NAT<br>NAT<br>仅主机|192.168.8.200/24<br>172.25.250.9/24<br>192.168.51.200/24|容器镜像仓库<br>Gitlab 代码库<br>Helm 服务器<br>DNS 服务器<br>Haproxy服务器<br>OpenShift 客户端||
|2|master01|8核心|16G内存，推荐20G|500G|NAT<br>仅主机|192.168.8.10/24<br>192.168.51.10/24|OpenShift v4.12|Multus 网络网卡名称 ens192<br>Multus IP 地址：192.168.51.10/24|

# 内容说明

所有虚拟机都做了一个名为v4.12的快照，有问题可以恢复到这个快照进行恢复，但是请注意，不要在开机状态下直接恢复快照，可能会导致虚拟机文件丢失

OpenShift、Quay和Gitlab的控制台书签已在student用户的浏览器就绪

1.	OpenShift 服务器

**不要试图ssh和控制台登录master01，OpenShift不支持这些登录方式，你只需要用oc命令连接**

> 1. 控制台URL：https://console-openshift-console.apps.ocp4.example.com
> 2. 临时登录用户名kubeadmin
> 3. 密码gdtpf-qvRqf-pekZF-2vABT
___

2.	Quay容器镜像仓库

> 1. URL：https://registry.ocp4.example.com:8443
> 2. 用户名：admin
> 3. 密码: redhatocp
___

3.	GitLab 代码仓库

> 1. URL：https://git.ocp4.example.com
> 2. 用户名：developer
> 3. 密码: redhatocp

# 虚拟机账号密码信息

虚拟机账号密码：

1. student用户名密码是student
2. root用户名密码是redhat

# 关于集群的开机和重启

包括集群虚拟机的第一次开机以及后续的每次重启，都会生成新的客户端证书请求，如果长时间集群无法ready，可能是当次的启动证书没有自动审批，执行以下命令完成证书审批即可

每次启动可能需要**15分钟**左右，耐心等待，在集群启动之前，这条命令将会失败，你可以多次重试，直到可以返回数据

```bash
oc get csr -o name | xargs oc adm certificate approve
```

正常输出如下：

```text
certificatesigningrequest.certificates.k8s.io/csr-4pjtx approved
certificatesigningrequest.certificates.k8s.io/csr-72flh approved
certificatesigningrequest.certificates.k8s.io/csr-8brdv approved
certificatesigningrequest.certificates.k8s.io/csr-jn4m8 approved
certificatesigningrequest.certificates.k8s.io/csr-prn6v approved
certificatesigningrequest.certificates.k8s.io/csr-q6h5l approved
certificatesigningrequest.certificates.k8s.io/csr-srm27 approved
certificatesigningrequest.certificates.k8s.io/system:openshift:openshift-authenticator-89tmt approved
certificatesigningrequest.certificates.k8s.io/system:openshift:openshift-authenticator-n5j4r approved
certificatesigningrequest.certificates.k8s.io/system:openshift:openshift-monitoring-f87sx approved
certificatesigningrequest.certificates.k8s.io/system:openshift:openshift-monitoring-vr8c8 approved
```

在每次重启虚拟机时，都建议在所有操作系统全起来之后执行上面这个命令，直到集群的所有operator可用状态为True

审批证书后，集群将会经过一阵繁忙的初始化操作，请耐心等待，直到执行下面的命令时，所有的集群operator都是正常

```bash
oc get co
```