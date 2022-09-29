```textile
作者：李晓辉

微信：Lxh_Chat

邮箱：xiaohui_li@foxmail.com
```

# 生成root证书信息

```bash
openssl genrsa -out /etc/pki/tls/private/xiaohuiroot.key 4096
openssl req -x509 -new -nodes -sha512 -days 3650 -subj "/C=CN/ST=Shanghai/L=Shanghai/O=Company/OU=SH/CN=xiaohui.cn" \
-key /etc/pki/tls/private/xiaohuiroot.key \
-out /etc/pki/ca-trust/source/anchors/xiaohuiroot.crt
```

# 生成服务器私钥以及证书请求文件

```bash
openssl genrsa -out /etc/pki/tls/private/xiaohui.cn.key 4096
openssl req -sha512 -new \
-subj "/C=CN/ST=Shanghai/L=Shanghai/O=Company/OU=SH/CN=xiaohui.cn" \
-key /etc/pki/tls/private/xiaohui.cn.key \
-out xiaohui.cn.csr
```

# 生成openssl cnf扩展文件

```bash
cat > certs.cnf << EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = *.xiaohui.cn
EOF
```

# 签发证书

```bash
openssl x509 -req -in xiaohui.cn.csr \
-CA /etc/pki/ca-trust/source/anchors/xiaohuiroot.crt \
-CAkey /etc/pki/tls/private/xiaohuiroot.key -CAcreateserial \
-out /etc/pki/tls/certs/xiaohui.cn.crt \
-days 3650 -extensions v3_req -extfile certs.cnf
```

# 信任根证书

```bash
update-ca-trust
```
