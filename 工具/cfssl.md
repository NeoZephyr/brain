ca 配置文件，用来配置证书的使用场景和具体参数

```bash
cfssl print-defaults config > config.json
```

signing：表示该证书可用于签名其它证书（生成的 ca.pem 证书中 CA=TRUE）
server auth：表示 client 可以用该证书对 server 提供的证书进行验证
client auth：表示 server 可以用该证书对 client 提供的证书进行验证
expiry：876000h，证书有效期设置为 100 年

```json
{
    "signing": {
        "default": {
            "expiry": "87600h"
        },
        "profiles": {
            "iam": {
                "expiry": "876000h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                  	"client auth"
                ]
            }
        }
    }
}
```

证书签名请求文件

```bash
cfssl print-defaults csr > csr.json
```

C：Country，国家
ST：State，省份
L：Locality (L) or City，城市
CN：Common Name，服务器从证书中提取该字段作为请求的用户名（User Name），浏览器使用该字段验证网站是否合法
O：Organization，服务器从证书中提取该字段作为请求用户所属的组（Group）
OU：Company division（or Organization Unit – OU0，部门 / 单位

不同证书 csr 文件的 CN、C、ST、L、O、OU 组合必须不同，否则可能出现 PEER'S CERTIFICATE HAS AN INVALID SIGNATURE 错误

```json
{
    "CN": "iam-ca",
    "key": {
        "algo": "ecdsa",
        "size": 256	
    },
    "names": [
        {
            "C": "CN",
            "ST": "ShangHai",
            "L": "ShangHai",
          	"O": "",
          	"OU": ""
        }
    ],
  	"ca": {
    	"expiry": "876000h"
    }
}
```

创建 CA 根证书和私钥

```bash
cfssl gencert -initca csr.json | cfssljson -bare ca
```

创建文件，私钥 ca-key.pem 和证书 ca.pem，还会生成证书签名请求 ca.csr，用于交叉签名或重新签名

查看 cert 和 csr 信息
```bash
cfssl certinfo -cert ca.pem
cfssl certinfo -csr ca.csr
```

api 证书签名请求，api-csr.json

```json
{
    "CN": "iam-api",
    "key": {
        "algo": "ecdsa",
        "size": 256	
    },
    "names": [
        {
            "C": "CN",
            "ST": "ShangHai",
            "L": "ShangHai",
          	"O": "",
          	"OU": "api"
        }
    ],
    "hosts": [
    	"127.0.0.1",
      "localhost",
      "api.com"
    ]
}
```

hosts 字段是用来指定授权使用该证书的 IP 和域名列表

生成证书和私钥

```bash
cfssl gencert
-ca=ca.pem
-ca-key=ca-key.pem
-config=ca-config.json
-profile=iam api-csr.json | cfssljson -bare api
```