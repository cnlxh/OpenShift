```textile
作者：李晓辉

微信联系：Lxh_Chat

联系邮箱: 939958092@qq.com
```

本次我们要完成的是：

1. 从内部的GitLab代码仓库拉取代码

2. 通过OpenShift 的S2I功能，构建镜像

3. 推送至我们的内部镜像仓库

4. 从内部镜像仓库拉取到业务服务器

5. 生成我们的业务

[内部代码仓库](https://gitee.com/cnlxh/openshift/blob/master/Configure-GitLab-On-Premises.md) 

[内部镜像仓库](https://gitee.com/cnlxh/openshift/blob/master/Internal-Registry-in-OpenShift.md) 

# 准备代码仓库

先根据代码仓库构建的方法，创建一个公开的php的代码仓库，然后把仓库克隆到本地

## 安装Git客户端

```bash
yum install git -y
```

克隆仓库到本地

```bash
git clone http://content.cluster1.xiaohui.cn:5000/root/php.git
cd php
```

表明自己的身份

```bash
git config --global user.name "Xiaohui Li"
git config --global user.email "cnlxh@xiaohui.cn"
```

## 创建index.php

此处我们模拟的是php业务，我们就用一个index.php来代码业务本身，这个页面如果上线，会生成一个只有hello lixiaohui的文字页面

```bash
cat > index.php <<EOF
<?php

echo "Hello lixiaohui\n";

?>
EOF
```

## 提交代码到仓库

在push时，请输入自己的账号密码

```bash
git add .
git commit -m 'push code'
git push
```

# 部署PHP应用

```bash
oc new-app http://content.cluster1.xiaohui.cn:5000/root/php.git --name=phptest
```

## 查询部署过程中创建的资源

```bash
[root@content php]# oc get buildconfigs
NAME      TYPE     FROM   LATEST
phptest   Source   Git    1
[root@content php]# oc get deployments
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
phptest   1/1     1            1           9m19s
[root@content php]# oc get pod
NAME                       READY   STATUS      RESTARTS   AGE
phptest-1-build            0/1     Completed   0          9m21s
phptest-765ccc6ddf-gcxrd   1/1     Running     0          7m15s
[root@content php]# oc get imagestream
NAME      IMAGE REPOSITORY                                    TAGS     UPDATED
phptest   registry.apps.cluster1.xiaohui.cn/default/phptest   latest   7 minutes ago
[root@content php]# oc get service
NAME         TYPE           CLUSTER-IP        EXTERNAL-IP                            PORT(S)             AGE
kubernetes   ClusterIP      192.168.0.1       <none>                                 443/TCP             19h
openshift    ExternalName   <none>            kubernetes.default.svc.cluster.local   <none>              19h
phptest      ClusterIP      192.168.197.192   <none>                                 8080/TCP,8443/TCP   9m28s
```

## 查询自动化构建过程

执行这一步，可以详细看到发生了这么几件事：

1. 从代码仓库clone代码

2. 检测到我们的代码类型为php，并pull了php的镜像

3. 生成了dockerfile并构建了容器镜像

4. 将容器镜像推送到内部仓库

5. 通过buildconfig部署了应用

```bash
oc logs -f buildconfig/phptest
```

## 对外界开放PHP应用

```bash
oc expose service phptest --hostname=lixiaohui.apps.cluster1.xiaohui.cn --name phptest
```

```bash
[root@content php]# oc get route
NAME      HOST/PORT                            PATH   SERVICES   PORT       TERMINATION   WILDCARD
phptest   lixiaohui.apps.cluster1.xiaohui.cn          phptest    8080-tcp                 None
```

## 测试PHP应用的访问

发现我们通过自定义的域名已经成功访问到了我们的应用

```bash
[root@content php]# curl lixiaohui.apps.cluster1.xiaohui.cn
Hello lixiaohui
```

# 业务版本更新

这里测试的是程序员又更新了代码，看看OpenShift重新构建业务的流程

## 更新业务代码

注意切换到你仓库中，我们是针对原有的index.php更新了它的内容

```bash
cat > index.php <<EOF
<?php

echo "Hello lixiaohui version 2\n";

?>
EOF
```

## 将新代码推送到仓库

```bash
git add .
git commit -m 'push code version 2'
git push
```

## 触发应用重新构建

```bash
oc start-build phptest
```

## 查询重新构建过程

我们发现以下几件事：

1. buildconfig版本变成了2

2. deployment 有一个短暂的2/1

3. pod 出现了一个phptest-2-build，并保留了第一个版本，以备回退

```bash
[root@content php]# oc get pod
NAME                       READY   STATUS      RESTARTS   AGE
phptest-1-build            0/1     Completed   0          16m
phptest-2-build            1/1     Running     0          60s
phptest-765ccc6ddf-gcxrd   1/1     Running     0          14m
[root@content php]# oc get buildconfigs
NAME      TYPE     FROM   LATEST
phptest   Source   Git    2
[root@content php]# oc get deployments
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
phptest   2/1     1            2           17m 
[root@content php]# oc get deployments
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
phptest   1/1     1            1           19m

[root@content php]# oc get pod
NAME                       READY   STATUS      RESTARTS   AGE
phptest-1-build            0/1     Completed   0          17m
phptest-2-build            0/1     Completed   0          94s
phptest-6b6c774b5b-g4g7w   1/1     Running     0          13s


```

## 访问重新构建后的应用

```bash
[root@content php]# curl lixiaohui.apps.cluster1.xiaohui.cn
Hello lixiaohui version 2
```
