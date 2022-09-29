```textile
作者：李晓辉
微信联系：Lxh_Chat
联系邮箱: 939958092@qq.com
```

# 用户认证授权介绍

OCP 对集群的控制手段大约如下几种：

1. 使用RBAC 对象（如规则、角色和绑定）将它们分配给用户

2. 通过namespace来控制对集群的访问

3. 控制 Pod 可以执行的操作，使用安全性上下文约束(SCC)访问的资源

无身份验证或身份验证无效的 API 请求会被看作为由 `anonymous` 系统用户发出的请求。身份验证后，策略决定用户被授权执行的操作

## 用户

默认情况下，集群中只有 `kubeadmin` 用户

用户类型：

1. 普通用户，例如`lixiaohui` ，`zhangsan`

2. 系统用户，`system:admin` `system:openshift-registry` `system:node:node1.example.com`

3. 服务用户，`system:serviceaccount:default:deployer` `system:serviceaccount:foo:builder`

### 查询现有用户

```bash
[root@content ~]# oc get users -A
```

### 集成HTPasswd

默认情况下，集群只有kubeadmin临时管理员，这是不够的，此时我们可以使用HTPasswd以secret的形式来存储用户和密码，需要注意的是，添加的用户需要先登陆一次，才能在授权中或oc get users找到此用户，不然会报告无此用户

1. 创建存储用户和密码的文件，这里创建了一个文件名为userlist，内含一个名为lixaohui，且密码为1的用户
   
   ```bash
   [root@content ~]# htpasswd -cb userlist lixiaohui 1
   Adding password for user lixiaohui
   [root@content ~]# cat userlist
   lixiaohui:$apr1$5mKts3R0$yIhOUsWwv/gbXlXAwqs9p1
   ```

2. 创建 HTPasswd secret，要使用 HTPasswd 身份提供程序，必须定义一个含有 HTPasswd 用户文件的 secret
   
   ```bash
   oc create secret generic htpass-secret --from-file=htpasswd=userlist -n openshift-config
   ```

3. 配置身份提供程序
   
   ```bash
   cat > htpasswd.yaml <<EOF
   apiVersion: config.openshift.io/v1
   kind: OAuth
   metadata:
     name: cluster
   spec:
     identityProviders:
     - name: HTPasswd
       mappingMethod: claim
       type: HTPasswd
       htpasswd:
         fileData:
           name: htpass-secret
   EOF
   ```
   
   ```bash
   oc apply -f htpasswd.yaml
   ```
   
   此时登录控制台后，在OAuth中，就可以看到用户身份供应商变成了HTPasswd，执行oc get co指令，等待authentication处理结束，可能没那么快，请多次执行并等待结束
   
   尝试使用新用户登录系统
   
   ```bash
   [root@content ~]# oc login -u lixiaohui
   Authentication required for https://api.cluster1.xiaohui.cn:6443 (openshift)
   Console URL: https://api.cluster1.xiaohui.cn:6443/console
   Username: lixiaohui
   Password:
   Login successful.
   
   You don't have any projects. You can try to create a new project, by running
   
       oc new-project <projectname>
   
   [root@content ~]# oc whoami
   lixiaohui
   ```

#### 添加HTPasswd用户

在增删改用户时，都需要oc get co查看进度

刚才我们新增了一个lixiaohui用户，此时重新用管理员登录，再新增一个zhansgan用户

```bash
[root@content ~]# oc login -u system:admin
Logged into "https://api.cluster1.xiaohui.cn:6443" as "system:admin" using existing credentials.

You have access to 67 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "default".
[root@content ~]# oc get secret htpass-secret -ojsonpath={.data.htpasswd} -n openshift-config | base64 --decode > userlist
[root@content ~]# cat userlist
lixiaohui:$apr1$qO2Nawr1$dZaf5Kbg4ff7TokMkNHmN1 

[root@content ~]# htpasswd -b userlist lisi 1
Adding password for user lisi
[root@content ~]# htpasswd -b userlist wangwu 1
Adding password for user wangwu
[root@content ~]# cat userlist
lixiaohui:$apr1$WIRI/9vT$Amd0eOB6i7ikK8P1Z8EvD0
zhangsan:$apr1$rkmS44Ch$MQ/GA2uI6bS1ovuOaW8rs1
lisi:$apr1$wzAOFUch$kGD.AJIH.EN0mqRcx8AFX.
wangwu:$apr1$XGCi0YH7$DGLcugSzsbx6vhIlZeFDI1

[root@content ~]# oc create secret generic htpass-secret --from-file=htpasswd=userlist --dry-run=client -o yaml -n openshift-config | oc replace -f -
secret/htpass-secret replaced
```

如果login的时候报告x509证书：未知的证书颁发机构，请执行下方的指令

```bash
 cp /openshift/auth/kubeconfig .kube/config
```

```bash
[root@content ~]# oc login -u zhangsan
Authentication required for https://api.cluster1.xiaohui.cn:6443 (openshift)
Console URL: https://api.cluster1.xiaohui.cn:6443/console
Username: zhangsan
Password:
Login successful.

You don't have any projects. You can try to create a new project, by running

    oc new-project <projectname> 
```

#### 删除HTPasswd用户

删除用户的同时还要删除其identify资源

```bash
[root@content ~]# oc delete users zhangsan;oc delete identities.user.openshift.io htpasswd:zhangsan
user.user.openshift.io "lixiaohui" deleted
identity.user.openshift.io "htpasswd:zhangsan" deleted 
```

## 组

使用组同时为多个用户授予权限，而不必单独授予用户权限

除了明确定义的组外，还有系统组或虚拟组，它们由集群自动置备

以下列出了最重要的默认虚拟组：

| 虚拟组                          | 描述                             |
| ---------------------------- | ------------------------------ |
| `system:authenticated`       | 自动与所有经过身份验证的用户关联。              |
| `system:authenticated:oauth` | 自动与所有使用 OAuth 访问令牌经过身份验证的用户关联。 |
| `system:unauthenticated`     | 自动与所有未经身份验证的用户关联。              |

### 新建testonly组

```bash
[root@content ~]# oc adm groups new testonly
group.user.openshift.io/testonly created
[root@content ~]# oc get groups
NAME       USERS
testonly

```

### 将用户添加到组

将lixiaohui用户添加到testonly组

```bash
[root@content ~]# oc adm groups add-users testonly lixiaohui
group.user.openshift.io/testonly added: "lixiaohui"

```

### 将用户从组中删除

```bash
[root@content ~]# oc adm groups remove-users testonly lixiaohui
group.user.openshift.io/testonly removed: "lixiaohui"

```

### 删除组

```bash
[root@content ~]# oc delete groups testonly
group.user.openshift.io "testonly" deleted
[root@content ~]# oc get groups
No resources found

```

## RBAC

可以使用集群角色和绑定来控制谁对OCP 平台本身和所有项目具有各种访问权限等级

授权通过使用以下几项来管理：

| 授权对象 | 描述                                        |
| ---- | ----------------------------------------- |
| 规则   | 一组对象上允许的操作集合。例如，用户或服务帐户能否`创建（create）` Pod |
| 角色   | 规则的集合。可以将用户和组关联或绑定到多个角色                   |
| 绑定   | 用户和/组与角色之间的关联                             |

控制授权的 RBAC 角色和绑定有两个级别：

| RBAC 级别 | 描述                                                      |
|:------- | ------------------------------------------------------- |
| 集群 RBAC | 对所有项目均适用的角色和绑定。*集群角色*存在于集群范围，*集群角色绑定*只能引用集群角色。          |
| 本地 RBAC | 作用于特定项目的角色和绑定。虽然*本地角色*只存在于单个项目中，但本地角色绑定可以*同时*引用集群和本地角色。 |

集群角色绑定是存在于集群级别的绑定。角色绑定存在于项目级别。集群角色 *view* 必须使用本地角色绑定来绑定到用户，以便该用户能够查看项目。只有集群角色不提供特定情形所需的权限集合时才应创建本地角色。

### 默认集群角色

| 默认集群角色             | 描述                                                             |
| ------------------ | -------------------------------------------------------------- |
| `admin`            | 项目管理者。如果在本地绑定中使用，则 `admin` 有权查看项目中的任何资源，并且修改项目中除配额外的任何资源。      |
| `basic-user`       | 此用户可以获取有关项目和用户的基本信息。                                           |
| `cluster-admin`    | 此超级用户可以在任意项目中执行任何操作。当使用本地绑定来绑定一个用户时，这些用户可以完全控制项目中每一资源的配额和所有操作。 |
| `cluster-status`   | 此用户可以获取基本的集群状态信息。                                              |
| `cluster-reader`   | 用户可以获取或查看大多数对象，但不能修改它们。                                        |
| `edit`             | 此用户可以修改项目中大多数对象，但无权查看或修改角色或绑定。                                 |
| `self-provisioner` | 此用户可以创建自己的项目。                                                  |
| `view`             | 此用户无法进行任何修改，但可以查看项目中的大多数对象。不能查看或修改角色或绑定。                       |

下方展示了集群角色、本地角色、集群角色绑定、本地角色绑定、用户、组和服务帐户之间的关系

![](https://access.redhat.com/webassets/avalon/d/OpenShift_Container_Platform-4.11-Authentication_and_authorization-zh-CN/images/052176e6fe02a52b8f683f060d917ad1/rbac.png)

### 授权

使用以下几项来评估授权：

**身份**

用户名以及用户所属组的列表

**操作**

你执行的操作。在大多数情况下，这由以下几项组成：

- **项目**：你访问的项目。项目是一种附带额外注解的 Kubernetes 命名空间，使一个社区的用户可以在与其他社区隔离的前提下组织和管理其内容
- **操作动词**：操作本身：`get`、`list`、`create`、`update`、`delete`、`deletecollection` 或 `watch`
- **资源名称**：你访问的 API 端点

**绑定**

绑定的完整列表，用户或组与角色之间的关联。

OpenShift Container Platform 通过以下几个步骤评估授权：

1. 使用身份和项目范围操作来查找应用到用户或所属组的所有绑定
2. 使用绑定来查找应用的所有角色
3. 使用角色来查找应用的所有规则
4. 针对每一规则检查操作，以查找匹配项
5. 如果未找到匹配的规则，则默认拒绝该操作

### 新增集群管理员

由于kubeadm是临时管理员，不方便协作，我们创建新的集群管理员，并删除kubeadm，刚才我们创建了lixiaohui用户，我们将其提升为集群级别的管理员

```bash
[root@content ~]# oc adm policy add-cluster-role-to-user cluster-admin lixiaohui
clusterrole.rbac.authorization.k8s.io/cluster-admin added: "lixiaohui"

```

注意下面的cluster-admin-0，这个是一个递增的数字，如果不确定，可以grep cluster-admin查询，一般上面的步骤成功之后，也不需要查询

```bash
[root@content ~]# oc describe clusterrolebindings cluster-admin-0
Name:         cluster-admin-0
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  ClusterRole
  Name:  cluster-admin
Subjects:
  Kind  Name       Namespace
  ----  ----       ---------
  User  lixiaohui

```

### 删除kubeadm临时管理员

我们目前已经有了集群管理员，删除kubeadm临时管理员，需要注意的是，如果没有别的cluster-admin用户，删除后，集群将需要重新安装

```bash
[root@content ~]# oc delete secrets kubeadmin -n kube-system
secret "kubeadmin" deleted
```

执行oc get co等待任务结束后，再次打开web控制台，就会发现kubeadm选项不见了

### 查询角色以及绑定

查看集群角色及其关联的规则集，必要时，请把get换成describe来查询其详情

```bash
[root@content ~]# oc get roles
[root@content ~]# oc get clusterrole.rbac
```

查看当前本地角色绑定集合，这显示绑定到当前项目的不同角色的用户和组

```bash
[root@content ~]# oc get rolebinding.rbac
```

要查其他项目的本地角色绑定，请向命令中添加 `-n` 标志

```bash
[root@content ~]# oc get rolebinding.rbac -n default
```

查看当前的集群角色绑定集合，必要时，请把get换成describe来查询其详情

```bash
[root@content ~]# oc get clusterrolebinding.rbac
```

## 项目和命名空间

项目是附带额外注解的 Kubernetes 命名空间，资源必须属于某个项目，不同项目的资源互不可见，用户必须由管理员授予对项目的访问权限；或者，如果用户有权创建项目，则自动具有自己创建项目的访问权限。

项目可以有单独的 `name`、`displayName` 和 `description` ，`openshift-` 开头的项目对用户而言最为重要。这些项目托管作为 Pod 运行的主要组件和其他基础架构组件

### 创建项目

```bash
[root@content ~]# oc new-project project1
```

### 授予用户对项目的权限

将lixiaohui设为project1的admin，如果想要设置为其他角色，请修改admin字符串，例如修改为view就是仅查看权限，另外，也需要注意我们执行的是add-role-to-user，执行的是本地角色绑定，请务必加-n来确定添加到指定的项目，也可以使用add-cluster-role-to-user，来添加集群范围内所有项目的权限，就不需要加-n，除了执行to-user之外，还可以执行to-group，以组的形式授权

```bash
[root@content ~]# oc adm policy add-role-to-user admin lixiaohui -n project1
```

```bash
[root@content ~]# oc get rolebinding.rbac -n project1
NAME                    ROLE                               AGE
admin                   ClusterRole/admin                  11m
admin-0                 ClusterRole/admin                  10m
system:deployers        ClusterRole/system:deployer        11m
system:image-builders   ClusterRole/system:image-builder   11m
system:image-pullers    ClusterRole/system:image-puller    11m 


[root@content ~]# oc describe rolebinding admin-0
Name:         admin-0
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  ClusterRole
  Name:  admin
Subjects:
  Kind  Name       Namespace
  ----  ----       ---------
  User  lixiaohui


```

### 撤掉用户对于项目的权限

同理，这里可以是remove-cluster-role，后面也可以from-group

```bash
[root@content ~]# oc adm policy remove-role-from-user admin lixiaohui -n project1
clusterrole.rbac.authorization.k8s.io/admin removed: "lixiaohui"

```


