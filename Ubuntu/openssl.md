

## Openssl ## 

### 证书

> 对证书所发布的公钥进行权威的认证，证书可以有效的避免中间人攻击的问题

```
1. PKC：Public-Key Certificate，公钥证书，简称证书。
2. CA：Certification Authority，认证机构。对证书进行管理，负责 1.生成密钥对、2. 注册公钥时对身份进行认证、3. 颁发证书、4. 作废证书。其中负责注册公钥和身份认证的，称为 RA（Registration Authority 注册机构）
3. PKI：Public-Key Infrastructure，公钥基础设施，是为了更高效地运用公钥而制定的一系列规范和规格的总称。比较著名的有PKCS（Public-Key Cryptography Standards，公钥密码标准，由 RSA 公司制定）、X.509 等。PKI 是由使用者、认证机构 CA、仓库（保存证书的数据库）组成。
CRL：Certificate Revocation List 证书作废清单，是 CA 宣布作废的证书一览表，会带有 CA 的数字签名。一般由处理证书的软件更新 CRL 表，并查询证书是否有效。
4.证书的编码格式:pem,der
```

1生成私钥

```shell
openssl genrsa -des3 -out server.key 1024
```

```sh
openssl genrsa [-out filename] [-passout arg] [-des] [-des3] [-idea] [-f4] [-3] [-rand file(s)] [-engine id] [numbits]
```



2.生成自签名证书

```sh
openssl req -out server.crt -new -key server.key -x509 -days 365 -batch
-new: 生成新证书签署请求
-x509: 专用于CA生成自签证书
-key: 生成请求时用到的私钥文件
-days n：证书的有效期限
-out /PATH/TO/SOMECERTFILE: 证书的保存路径=
```

3.从私钥中删除密码

1. `cp server.key server.key.secure`

2. `openssl rsa -in server.key.secure -out server.key`

   

Once you've done that, you need to change the filenames in server.cpp and client.cpp.

server.cpp

1. `context_.use_certificate_chain_file("server.crt"); `
2. `context_.use_private_key_file("server.key", boost::asio::ssl::context::pem);`
3. `//context_.use_tmp_dh_file("dh512.pem");`

client.cpp

```
ctx.load_verify_file("server.crt");
```