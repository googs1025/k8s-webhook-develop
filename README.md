## k8s api server webhook 二次开发实践
## k8s-webhook-develop

### 项目思路
#### 1. Validate
使用k8s插件的webhook功能，部署ValidatingWebhookConfiguration资源对象。

同时支持白名单与黑名单校验镜像的两种方式。

(在yaml/deploy.yaml的环境变量WhITE_OR_BLOCK中设置"white" or "black"。)

1. 白名单：只有列表中的镜像前缀同意创建。

2. 黑名单：只有列表中的镜像前缀拒绝创建。
#### 2. Mutate
使用k8s插件的webhook功能，部署MutatingWebhookConfiguration资源对象。

支持替换pod image功能或增加annotation功能

(在yaml/deploy.yaml的环境变量ANNOTATION_OR_IMAGE中设置"image" or "annotation"。)

// **TODO**: 目前仅支持替换镜像patch功能，未来预计新增类似**istio**的注入特定容器功能。
### 项目部署步骤
1. 进入目录

2. 编译镜像
```
docker run --rm -it -v /root/k8sWebhookPractice:/app -w /app -e GOPROXY=https://goproxy.cn -e CGO_ENABLED=0  golang:1.18.7-alpine3.15 go build -o ./webhookPractice .
```

3. 使用cfssl 建立证书
a.下载CA证书软件
```
# Linux
➜  wget -q --show-progress --https-only --timestamping \
  https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 \
  https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
➜  chmod +x cfssl_linux-amd64 cfssljson_linux-amd64
➜  sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl
➜  sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
```
b. 创建 CA 证书
```
➜  cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "server": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

➜  cat > ca-csr.json <<EOF
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF

➜  cfssl gencert -initca ca-csr.json | cfssljson -bare ca
2021/01/23 16:59:51 [INFO] generating a new CA key and certificate from CSR
2021/01/23 16:59:51 [INFO] generate received request
2021/01/23 16:59:51 [INFO] received CSR
2021/01/23 16:59:51 [INFO] generating key: rsa-2048
2021/01/23 16:59:51 [INFO] encoded CSR
2021/01/23 16:59:51 [INFO] signed certificate with serial number 502715407096434913295607470541422244575186494509
➜  ls -la *.pem
-rw-------  1 ych  staff  1675 Jan 23 17:05 ca-key.pem
-rw-r--r--  1 ych  staff  1310 Jan 23 17:05 ca.pem

```
c.
```
➜  cat > server-csr.json <<EOF
{
  "CN": "admission",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
        "C": "CN",
        "L": "BeiJing",
        "ST": "BeiJing",
        "O": "k8s",
        "OU": "System"
    }
  ]
}
EOF

➜  cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json \
		-hostname=admission-registry.default.svc -profile=server server-csr.json | cfssljson -bare server
2021/01/23 17:08:37 [INFO] generate received request
2021/01/23 17:08:37 [INFO] received CSR
2021/01/23 17:08:37 [INFO] generating key: rsa-2048
2021/01/23 17:08:37 [INFO] encoded CSR
2021/01/23 17:08:37 [INFO] signed certificate with serial number 701199816701013791180179639053450980282079712724
➜  ls -la *.pem
-rw-------  1 ych  staff  1675 Jan 23 17:05 ca-key.pem
-rw-r--r--  1 ych  staff  1310 Jan 23 17:05 ca.pem
-rw-------  1 ych  staff  1675 Jan 23 17:08 server-key.pem
-rw-r--r--  1 ych  staff  1452 Jan 23 17:08 server.pem

```

4.使用生成的 server 证书和私钥创建一个 Secret 对象

```
# 创建Secret
➜  kubectl create secret tls admission-registry-tls \
        --key=server-key.pem \
        --cert=server.pem
secret/admission-registry-tls created
```


5. 启动项目(目前支援两个其中一个部署，如果同时部署会报错)
```
# 如果使用validate webhook
kubectl apply -f deploy.yaml
kubectl apply -f validatewebhook.yaml
# 如果使用 mutate webhook
kubectl apply -f deploy.yaml
kubectl apply -f mutatewebhook.yaml

[root@vm-0-12-centos yaml]# kubectl apply -f .
deployment.apps/admission-registry created
service/admission-registry created
validatingwebhookconfiguration.admissionregistration.k8s.io/admission-registry created
```

6. 测试

test.yaml : 主要测试validate webhook 通过白名单的前缀

test1.yaml：主要测试validate webhook 非白名单的前缀 会报错，也可以测试mutate webhook 更换镜像后不会报错。

test3.yaml：主要测试mutate webhook 新增annotation
```
kubectl apply -f test.yaml
kubectl apply -f test1.yaml
kubectl apply -f test3.yaml
```

7. deployment配置

**重点**

目录：yaml/deploy.yaml，主要说明配置文件的环境变量
   
```bigquery
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admission-registry
  labels:
    app: admission-registry
spec:
  replicas: 1
  selector:
    matchLabels:
      app: admission-registry
  template:
    metadata:
      labels:
        app: admission-registry
    spec:
      containers:
        - name: validate
          image: alpine:3.12
          imagePullPolicy: IfNotPresent
          command: ["/app/webhookPractice"]
          env: # 环境变量
            # 前三个是validate webhook使用
            - name: WHITELIST_REGISTRIES # 白名单列表
              value: "docker.io,gcr.io"
            - name: BLACKLIST_REGISTRIES # 黑名单列表
              value: ""
            - name: WhITE_OR_BLOCK # 使用白名单还是黑名单
              value: "white"
            - name: PORT # 端口
              value: "443"
            # 后三个是mutate webhook使用
            - name: ANNOTATION_OR_IMAGE # 判断mutate是patch镜像功能还是补全annotation
              value: "annotation"
            - name: MUTATE_PATCH_IMAGE # 替换的容器镜像
              value: "nginx:1.19-alpine"
            - name: IS_INIT_IMAGE # 是否使用initContainer容器 (测试阶段)
              value: "true"
          ports:
            - containerPort: 443
          volumeMounts:
            - name: webhook-certs
              mountPath: /etc/webhook/certs # 需要注意上面步骤的密钥文件是否成功创建并在主机中此目录
              readOnly: true
            - name: app
              mountPath: /app
      volumes:
        - name: webhook-certs
          secret: # 把secret 映射到volumes。 最终会转为tls.crt tls.key
            secretName: admission-registry-tls # 可以先使用此检查看看 kubectl get secret | grep admission-registry-tls
        - name: app
          hostPath:
            path: /root/k8sWebhookPractice # 可执行文件的挂载目录
```
