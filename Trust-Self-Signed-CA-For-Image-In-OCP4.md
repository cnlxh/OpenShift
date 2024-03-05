```textile
作者：李晓辉

微信：Lxh_Chat

邮箱：xiaohui_li@foxmail.com
```

|OCP版本|QUAY版本|QUAY自签名|QUAY地址|
|-|-|-|-|
|4.12|3.8.14|是|registry.ocp4.example.com:8443|


本文主要解决在OpenShift 4版本中，无法从第三方外部仓库中拉取镜像的问题，无法拉取镜像的原因是QUAY是自签名证书，所以在OpenShift中不信任是正常情况，以下是解决思路和步骤：

先完成RHCOS操作系统登录

```bash
ssh core@master01
```

以下所有操作都在RHCOS的OpenShift节点

1. 让操作系统信任自签名CA证书

我的自签名CA证书名称是：`xiaohuiroot.crt`

```bash
cp xiaohuiroot.crt /etc/pki/ca-trust/source/anchors/
update-ca-trust
```
2. 登录OpenShift

用管理员admin登录openshift

```bash
oc login -u admin -p redhatocp https://api.ocp4.example.com:6443
```

3. 创建一个configmap

这个configmap用于稍后让openshift来引用ca内容，经过第一个步骤后，我的CA证书位于: `/etc/pki/ca-trust/source/anchors/xiaohuiroot.crt`

如果你的镜像仓库是默认的https标准端口443，就用以下命令

```bash
oc create configmap registry-config --from-file=registry.ocp4.example.com=/etc/pki/ca-trust/source/anchors/xiaohuiroot.crt -n openshift-config
```

如果你的镜像仓库是`不是`默认的https标准端口443，就用以下命令，本次我的镜像仓库端口是`8443`

注意，这里的端口形式必须写成`..8443`

```bash
oc create configmap registry-config --from-file=registry.ocp4.example.com..8443=/etc/pki/ca-trust/source/anchors/xiaohuiroot.crt -n openshift-config
```

4. 确认configmap符合预期

```bash
oc describe configmaps -n openshift-config registry-config
```

```yaml
Name:         registry-config
Namespace:    openshift-config
Labels:       <none>
Annotations:  <none>

Data
====
registry.ocp4.example.com..8443:
----
-----BEGIN CERTIFICATE-----
MIIFrzCCA5egAwIBAgIUdvbmpaLMXoTKklC9iIJTn8uQf0swDQYJKoZIhvcNAQEN
BQAwZzELMAkGA1UEBhMCQ04xETAPBgNVBAgMCFNoYW5naGFpMREwDwYDVQQHDAhT
aGFuZ2hhaTEQMA4GA1UECgwHQ29tcGFueTELMAkGA1UECwwCU0gxEzARBgNVBAMM
CnhpYW9odWkuY24wHhcNMjQwMjI4MDgzNzIyWhcNMzQwMjI1MDgzNzIyWjBnMQsw
CQYDVQQGEwJDTjERMA8GA1UECAwIU2hhbmdoYWkxETAPBgNVBAcMCFNoYW5naGFp
MRAwDgYDVQQKDAdDb21wYW55MQswCQYDVQQLDAJTSDETMBEGA1UEAwwKeGlhb2h1
aS5jbjCCAiIwDQYJKoZIhvcNAQEBBQADggIPADCCAgoCggIBALauxPD3q4JoehR6
/B5jxN2kVdpNC7XcKQZu4BoQnQHcCMQLRgEsMIwk2FzbMmgfMpZmIrBwAAl67auB
G9lKAvWW1vxJTghyRMY8B2yEgTNu35Spa/XEKqyW776jYWaEmiOMzyX0zqcNa0jN
O3Dy5R+JaUAOOEtkPXsfSphDDX98/N6XqGh+OnCa2ZoPiVnp0FxbVJk2u24Kc5Bk
D50OdjYFFndTrFrAIGsBl3lyR8GBB9Qw9d0SL8girG0aifLP2dpyeqlNvuesbAQe
xU6tOt+cPB3vVx8GRffDDG8wLaFkCaFeVCyU6ytHPvk2iltchPMmOGPQCdQX7sAa
5Lny9wSCWnl9QSC3uxbGaE5uE2u/oHAbpW7tNvF7gGk34gDKMydoPbatyxMocKJQ
2XHytt1K7yZx6EsREwpVTV5WG1l56RTxyAHeXzItt1znom0kiOOL4tJmaUpOETqM
IQH034GQK1LjboH71Q9WwdHJilRqnnU8LWgHKtuib0a+zjSQUwuE3y26XmslWS35
isO+7htn7n1bgTLwC7om6EsocAPfhOlbILH2QXjsoWC+g0YYFNAUN4KDjPsHdCT1
gUjod60WBU6zuiQvOEBBnpIaEDdKmbk2GZSF2yVkGUhnyezKxPmVZo7q8jHlT9F/
TVnaqhiS7FeQ20IqiVkons4v4rZnAgMBAAGjUzBRMB0GA1UdDgQWBBSVeiXlJH36
1GBbrsWC8J/Iel5wqzAfBgNVHSMEGDAWgBSVeiXlJH361GBbrsWC8J/Iel5wqzAP
BgNVHRMBAf8EBTADAQH/MA0GCSqGSIb3DQEBDQUAA4ICAQAq3mzc84ptBvdik9GT
+LS7At99Kaeox4jMJxvArwWddbwZPedqISMC0UcgHQ8kCZZW7BbJ95pC15SPP6E0
GULimBFFSFMnM4MQGzze4bPTBdcLCGTGWJ2vMrihZ1SV1nhb/S79MryL6VEWS39G
mi71CepNT/Oa2qIbUE6sXL760YhrIHq/3Qk3rfIQkqzRObGuchE/atRU66tDwKJs
TllUYOMgJExgajy3U/2BiL7wfsPTGun9K1JziNy7Q3iYru1Z/H2+eDH4mI+QxJDB
bL+iGygevlA85i/6gJMVXQNBY/Yxnfqj+kT14/UXzuW+BAroWO7CExW4tm3XFvi7
73JRFFaz6M3/80CVEc2yxM0M8KR8EO2NZ9nAhqXMT1JmM7QI4J7G30JqVuZ80S5Q
F9Q2QrEW55VpGaJ/5Ikxh5ZPLpuMYXSTONA5+Giam+opt3xS+jeAFcIuOAa5UhLW
0HbxIlntm7jdbfl2pdPTvpsLwXxS4bu7DVJkgtjsp4KCTON4mTvq79D+Ra7QLOlV
SWyLrKRODNwKu/tiIz4d/qQANha2jHD3D1ipjdfVJIuqsNwt9FxBiaCHXkSSuskf
aDzEzp+ZjJLUwaFflB+AuTlRUFSKbTg480F1Ftgb2lVTsZBerGnoAtyYV/XrgQFD
35YDAHb/q0tk5ghFo5paAGvwFg==
-----END CERTIFICATE-----


BinaryData
====

Events:  <none>
```

5. 信任额外的CA证书颁发机构

```bash
oc edit image.config.openshift.io cluster
```

只在spec下添加了两行

```yaml
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: config.openshift.io/v1
kind: Image
...
spec:
  additionalTrustedCA:
    name: registry-config
```

6. 验证信任结果

这个需要稍微等待几分钟，在此期间可以执行以下命令来查看变化

```bash
oc get co
```

测试从我们自签名的镜像仓库中部署应用

```bash
 oc new-app --name hello --image registry.ocp4.example.com:8443/redhattraining/hello-world-nginx:v1.0
```

```bash
 --image registry.ocp4.example.com:8443/redhattraining/hello-world-nginx:v1.0
--> Found container image 44eaa13 (4 years old) from registry.ocp4.example.com:8443 for "registry.ocp4.example.com:8443/redhattraining/hello-world-nginx:v1.0"

    Red Hat Universal Base Image 8
    ------------------------------
    The Universal Base Image is designed and engineered to be the base layer for all of your containerized applications, middleware and utilities. This base image is freely redistributable, but Red Hat only supports Red Hat technologies through subscriptions for Red Hat products. This image is maintained by Red Hat and updated regularly.

    Tags: base rhel8

    * An image stream tag will be created as "hello:v1.0" that will track this image

--> Creating resources ...
    imagestream.image.openshift.io "hello" created
    deployment.apps "hello" created
    service "hello" created
--> Success
    WARNING: No container image registry has been configured with the server. Automatic builds and deployments may not function.
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose service/hello'
    Run 'oc status' to view your app.
```