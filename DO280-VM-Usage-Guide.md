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

请确保workstation已经正常开机看到桌面后，再开master01

**请勿修改虚拟机中的任何网咯信息，日常练习执行命令都在workstation这一台机器上**

|开机顺序|主机名|CPU|内存|硬盘|VMware网卡类型|IP|角色|备注|
|:-:|:-:|:-:|:-:|:-:|:-|:-|:-|:-|
|1|workstation|8核心|16G内存，推荐20G|500G|NAT<br>NAT<br>仅主机|192.168.8.200/24<br>172.25.250.9/24<br>192.168.51.200/24|容器镜像仓库<br>Gitlab 代码库<br>Helm 服务器<br>DNS 服务器<br>Haproxy服务器<br>OpenShift 客户端||
|2|master01|8核心|16G内存，推荐20G|500G|NAT<br>仅主机|192.168.8.10/24<br>192.168.51.10/24|OpenShift v4.12|Multus 网络网卡名称 ens192<br>Multus IP 地址：192.168.51.10/24|

# 内容说明

所有虚拟机都做了一个名为v4.12的快照，有问题可以恢复到这个快照进行恢复，但是请注意，不要在开机状态下直接恢复快照，可能会导致虚拟机文件丢失

OpenShift、Quay和Gitlab的控制台书签已在student用户的浏览器就绪

1.	OpenShift 服务器

> 控制台URL：https://console-openshift-console.apps.ocp4.example.com

> 临时登录用户名kubeadmin，密码gdtpf-qvRqf-pekZF-2vABT
___

2.	Quay容器镜像仓库

URL：https://registry.ocp4.example.com:8443

> 1. 用户名：admin
> 2. 密码: redhatocp
___

3.	GitLab 代码仓库

> 1. URL：https://git.ocp4.example.com
> 2. 用户名：developer
> 3.密码: redhatocp

# 其他信息

虚拟机账号密码：

1. student用户名密码是student
2. root用户名密码是redhat

# 关于集群的开机和重启

包括集群虚拟机的第一次开机以及后续的每次重启，都会生成新的客户端证书请求，如果长时间集群无法ready，可能是当次的启动证书没有自动审批。在workstation上用oc login登录OCP集群之后，执行以下命令完成证书审审批即可

```bash
oc get csr -o name | xargs oc adm certificate approve
```
