```textile
作者：李晓辉
微信联系：Lxh_Chat
联系邮箱: 939958092@qq.com
```

# 切换Registry状态

在例如裸机等没有可共享对象存储的平台上安装时， Image Registry Operator 配置是Removed，我们必须将`managementState` 从 `Removed` 切换到 `Managed` ，内部仓库才可以使用，虽然也可以搭建第三方 registry，但是可能不会支持镜像通知

```bash
oc patch configs.imageregistry.operator.openshift.io cluster \
--type merge --patch '{"spec":{"managementState":"Managed"}}'
```

# 创建PV

这里采用了NFS的方法，生产环境请使用可靠的存储，例如Ceph等

## 创建nfs资源

```bash
[root@content ~]# yum install nfs-utils -y
[root@content ~]# mkdir /nfs
[root@content ~]# semanage fcontext -a -t public_content_t "/nfs(/.*)?"
[root@content ~]# restorecon -RvF /nfs/
[root@content ~]# chmod 777 /nfs/ -R
[root@content ~]# systemctl enable nfs-server --now
```

```bash
cat >> /etc/exports <<EOF
/nfs *(rw)
EOF
systemctl restart nfs-server
firewall-cmd --add-service=rpc-bind --add-service=nfs --add-service=mountd --permanent
firewall-cmd --reload
```

## 创建PV

```bash
cat > pv.yaml <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: internal-registry
spec:
  capacity:
    storage: 200Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /nfs
    server: 172.16.50.200 
EOF

oc apply -f pv.yaml
```

## 创建PVC

这里的PVC其实也可以不建立，只需要在下一步的spec.storage下，将pvc的claim中保持空值，就会自动创建PVC，自己创建PVC可以定制名称等参数

```bash
cat > pvc.yml <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: internal-registry-pvc
  namespace: openshift-image-registry
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 200Gi
EOF
oc apply -f pvc.yaml
```

# 配置Registry后端存储

```bash
oc edit configs.imageregistry.operator.openshift.io
```

将spec下的storage改成我们的pvc，并保存退出，然后执行oc get co，等待它自己执行结束

```yaml
spec:
...
  storage:
    pvc:
      claim: internal-registry-pvc
```

# 验证Registry Pod状态

在配置后端存储之前执行这个命令会报告没有pod资源，配置后，就会有一个pod了

```bash
oc get pod -n openshift-image-registry -l docker-registry=default
```

# 启用镜像的构建和推送

```bash
oc edit configs.imageregistry/cluster
```

在spec.storage中将managementState状态改一下

```yaml
spec:
...
  storage:
    managementState: Managed
    pvc:
      claim: internal-registry-pvc
```

# 访问Registry

由于这是内部registry，所以需要从集群内进行访问，在content机器上执行下方指令进行集群内命令执行，注意主机名

```bash
 oc debug nodes/master0.cluster1.xiaohui.cn

 chroot /host
```

登录OCP和registry，在podman login的时候，用户名可以是任何字符串，不需要是真的用户名，此时用的是token

```bash
oc login -u lixiaohui -p 1 https://api.cluster1.xiaohui.cn:6443
podman login -u lixiaohui -p $(oc whoami -t) image-registry.openshift-image-registry.svc:5000
```

搜索registry内容，注意后面的斜杠/，此时会列出很多镜像

```bash
podman search image-registry.openshift-image-registry.svc:5000/
```

拉取镜像，从别处拉取镜像，尝试推送到我们的仓库中

```bash
podman pull docker.io/hello-world
```

将镜像改名，需要注意的是，这个名称格式是fqdn:5000/projectname/imagename:tag

```bash
 podman tag docker.io/hello-world image-registry.openshift-image-registry.svc:5000/openshift/hello-world
```

推送镜像到仓库中，我没有执行tag，那默认就会是latest，如果推送失败，可以尝试将自己关联system:image-builder这个系统角色

```bash
podman push image-registry.openshift-image-registry.svc:5000/openshift/hello-world
```

删除仓库中的镜像

```bash
[root@content ~]# oc get images | grep hello
sha256:e2ce951141b668014b7ea85b4167c957593763227b7a121938002532b29d9dbd   image-registry.openshift-image-registry.svc:5000/openshift/hello-world@sha256:e2ce951141b668014b7ea85b4167c957593763227b7a121938002532b29d9dbd
```

看到镜像的名字是sha256...，复制一下，然后删除

```bash
oc delete images sha256:e2ce951141b668014b7ea85b4167c957593763227b7a121938002532b29d9dbd
```

# 公开Registry

通过使用路由可以开放从外部访问Registry

如果要自动启用Image Registry默认路由，对 Image Registry Operator CRD 进行 patch 处理，然后执行oc get co，等待执行结束

```bash
oc patch configs.imageregistry.operator.openshift.io/cluster \
--type merge -p '{"spec":{"defaultRoute":true}}'
```

获取默认 registry 路由

```bash
HOST=$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')
```

获取 Ingress Operator 的证书

```bash
oc get secret -n openshift-ingress  router-certs-default -o go-template='{{index .data "tls.crt"}}' | \
base64 -d | sudo tee /etc/pki/ca-trust/source/anchors/${HOST}.crt  > /dev/null
```

使用以下命令启用集群的默认证书来信任路由

```bash
update-ca-trust
```

使用默认路由登录

```bash
podman login -u lixiaohui -p $(oc whoami -t) $HOST
```

如果登录的时候报告如下错误

```bash
error: no token is currently in use for this session
```

登录一下ocp，然后再登录podman就可以了

```bash
oc login -u lixiaohui -p 1 https://api.cluster1.xiaohui.cn:6443
```

# 自定义route路径

默认情况下，registry的route域名太长了，不方便，我们可以定制一下，但是这个是TLS加密的，所以我们需要重新申请证书

根证书生成

```bash
openssl genrsa -out /etc/pki/tls/private/xiaohuiroot.key 4096
openssl req -x509 -new -nodes -sha512 -days 3650 -subj "/C=CN/ST=Shanghai/L=Shanghai/O=Company/OU=SH/CN=xiaohui.cn" \
-key /etc/pki/tls/private/xiaohuiroot.key \
-out /etc/pki/ca-trust/source/anchors/xiaohuiroot.crt 
```

生成服务器私钥以及证书请求文件

```bash
openssl genrsa -out /etc/pki/tls/private/registry.key 4096
openssl req -sha512 -new \
-subj "/C=CN/ST=Shanghai/L=Shanghai/O=Company/OU=SH/CN=xiaohui.cn" \
-key /etc/pki/tls/private/registry.key \
-out registry.csr 
```

生成openssl cnf扩展文件

```bash
cat > certs.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = registry.apps.cluster1.xiaohui.cn
EOF
```

签发证书

```bash
openssl x509 -req -in registry.csr \
-CA /etc/pki/ca-trust/source/anchors/xiaohuiroot.crt \
-CAkey /etc/pki/tls/private/xiaohuiroot.key -CAcreateserial \
-out /etc/pki/tls/certs/registry.crt \
-days 3650 -extensions v3_req -extfile certs.cnf
```

创建Secret

```bash
oc create secret tls registry -n openshift-image-registry \
--cert=/etc/pki/tls/certs/registry.crt \
--key=/etc/pki/tls/private/registry.key
```

创建route，新增route字段

```bash
oc edit configs.imageregistry.operator.openshift.io
```

```yaml
spec:
  routes:
    - name: public-routes
      hostname: registry.apps.cluster1.xiaohui.cn
      secretName: registry
```

此时我们就可以从外部通过registry.apps.cluster1.xiaohui.cn来完成image tag和image push了
