```
#openssl是一个由C语言写的开源的密码学工具包。
dnf install -y openssl
```



公钥和私钥怎么转为明文？

密钥文件本身是 Base64 编码（PEM）或二进制（DER），`-----BEGIN ... -----` 中间那段 Base64 是结构化数据，要靠 openssl 才能解析成可读字段。"转明文"通常是指用 openssl 解析出里面的结构化信息：

```
# 查看私钥内容（模数、指数等）（PEM 文件中间那段 Base64 本身不是"密文",只是把二进制 DER 数据编码成了文本,任何人都能解码。它不需要"解密"。）
openssl rsa -in private.key -text -noout

# 查看公钥
openssl rsa -in public.key -pubin -text -noout

# 解密带密码的私钥（这里的带密码是指的私钥文件用密码加密过(有Proc-Type: ENCRYPTED)）
openssl rsa -in encrypted.key -out decrypted.key

# DER（二进制）转 PEM（Base64 文本）
openssl pkey -inform der -in key.der -outform pem -out key.pem

# 从私钥提取公钥（PEM 里物理上就存着公钥,同时数学上私钥本来就能算出公钥）
openssl rsa -in xx.pem -pubout -out public.pem
```

```
openssl rsa -in xxx.pem -text -noout的各字段含义：

RSA Private-Key: (2048 bit, 2 primes)：这是一个 2048 位的 RSA 私钥，由 2 个质数生成。
modulus (n)：模数，等于两个大质数的乘积（prime1 × prime2），是公钥和私钥共享的部分。
publicExponent (e)：公钥指数，65537 是 RSA 的标准值。
privateExponent (d)：私钥指数，加密/签名运算的核心秘密。
prime1 (p) / prime2 (q)：生成模数的两个大质数，绝对机密。
exponent1 / exponent2 / coefficient：用于中国剩余定理（CRT）加速 RSA 运算的预计算值，也属于私钥的一部分。

命令最后的 -noout 表示不输出原始 PEM 编码内容，只显示解析后的数学参数。
```



crt 和 pem 的区别？

这俩主要是扩展名的差异，不是格式差异，经常被混用。PEM 是一种编码格式：Base64 + `-----BEGIN XXX-----` / `-----END XXX-----` 头尾。可以装证书、私钥、CSR、证书链等任何东西。CRT/CER 是用途标识：表示这是个证书文件。内容可以是 PEM（文本），也可以是 DER（二进制）。

经验法则:Linux 上的 `.crt` 基本都是 PEM 格式;Windows 导出的 `.cer` 经常是 DER 二进制。用 `file` 命令或者文本编辑器打开看一眼就知道——能看到 `BEGIN CERTIFICATE` 就是 PEM。

互转：

```
# DER → PEM
openssl x509 -inform der -in cert.crt -out cert.pem

# PEM → DER  
openssl x509 -in cert.pem -outform der -out cert.crt
```



域名证书的 PEM 里有什么？

一个用于部署的域名证书 PEM 文件（通常叫 fullchain.pem）一般是多张证书拼在一起：

```
-----BEGIN CERTIFICATE-----
（域名证书，end-entity / leaf cert）
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
（中间 CA 证书，intermediate）
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
（有时还有第二张中间证书）
-----END CERTIFICATE-----
```

根证书一般不放进去，因为客户端（浏览器/系统）已内置。Let's Encrypt 签发的证书包含这些文件：

- `cert.pem` — 域名证书
- `chain.pem` — 中间 CA 链
- `fullchain.pem` — 上面两个拼接（nginx 用这个）
- `privkey.pem` — 私钥（单独保存，不会和证书放一起）

```
openssl x509 -in scs1738827573590__.zeyqaq.com_server.crt -text -noout
 
所含内容：
基本信息
证书版本：v3
序列号：3b:7f:2c:41:44:c9:83:cc:d4:c0:1b:8b:f9:46:7d:d9
签名算法：sha384WithRSAEncryption

颁发者（Issuer）
国家：CN（中国）
组织：DNSPod, Inc.（腾讯云旗下）
CN：DNSPod RSA DV（域名验证型证书）

使用者（Subject）
CN：*.zeyqaq.com（通配符证书）

有效期

生效时间：2025 年 2 月 6 日
到期时间：2026 年 2 月 13 日

算法：RSA
长度：2048 位
指数：65537

关键扩展项
密钥用途：数字签名、密钥加密
扩展密钥用途：TLS 服务器认证、TLS 客户端认证
是否为 CA 证书：否（CA:FALSE，是终端实体证书）
覆盖域名（SAN）：*.zeyqaq.com

其他
CA 颁发地址：http://crt.trust-provider.cn/DNSPodRSADV.crt
OCSP 校验地址（OCSP Responder是给客户端去查吊销状态用的）：http://ocsp.trust-provider.cn
CT 日志（SCT）：包含 3 条证书透明度日志记录，用于公开审计
证书策略：包含 DV（域名验证）级别的策略 OID（2.23.140.1.2.1）
```



服务器证书本身包含什么

服务器证书是一个 X.509 结构，主要字段有：

- Subject：CN（Common Name，主域名）、O（组织）、C（国家）等
- Subject Alternative Name (SAN)：实际生效的域名列表，比如 `example.com`、`*.example.com`、`www.example.com`。现代浏览器只看 SAN，不再看 CN。
- 公钥：服务器的公钥（私钥不在里面！私钥永远只在服务器上）
- Issuer：签发它的 CA 信息
- 有效期：Not Before / Not After
- 序列号、签名算法（如 SHA256-RSA、ECDSA）
- 扩展字段：Key Usage、Extended Key Usage（TLS Web Server Authentication）、CRL 分发点、OCSP 地址等
- CA 的签名：用 CA 私钥对上面所有内容的签名，客户端用 CA 公钥验证

所有可信 CA 都在这里：

```
#Debian：
ls /etc/ssl/certs/
ls /usr/share/ca-certificates/
#CentOS：
ls /etc/pki/ca-trust/source/anchors/
ls /etc/pki/tls/certs/
#MacOS：
security find-certificate -a -p /Library/Keychains/System.keychain 
#Windows：
Get-ChildItem -Path Cert:\LocalMachine\Root      # 受信任的根
Get-ChildItem -Path Cert:\LocalMachine\My        # 个人证书
Get-ChildItem -Path Cert:\CurrentUser\My         # 当前用户的个人证书
```

```
华为云ECS虚拟机的某个证书：
身份：ACCVRAIZ1 根证书

颁发机构：ACCV（Autoritat de Certificació de la Comunitat Valenciana，西班牙瓦伦西亚自治区认证局）
国家：ES（西班牙）
这是一张自签名根证书（Issuer 和 Subject 相同）

有效期：2011-05-05 至 2030-12-31
密钥与签名：

RSA 4096 位公钥（强度够）
签名算法是 sha1WithRSAEncryption（SHA-1 目前已被认为不够安全，但对根证书的影响较小，因为根证书是靠"被预置在信任库里"而不是靠签名被验证来建立信任）

关键扩展字段：

CA:TRUE（critical）：是一张 CA 证书，可以签发下级证书
Key Usage：Certificate Sign, CRL Sign（只能用于签名证书和吊销列表）
OCSP 地址：http://ocsp.accv.es
CRL 地址：accv.es 上的吊销列表
联系邮箱：accv@accv.es
```



如何用nginx配置一个最小的https服务？

最小可用配置：

```nginx
server {
    listen 443 ssl;
    http2 on;
    server_name example.com www.example.com;

    ssl_certificate     /etc/nginx/ssl/fullchain.pem;   # 含完整证书链
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;     # 私钥，权限 600

    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
    ssl_prefer_server_ciphers off;

    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    # HSTS（可选，但谨慎开启，开了就难撤回）
    add_header Strict-Transport-Security "max-age=63072000" always;

    # OCSP Stapling（可选，提升握手性能）
    ssl_stapling on;
    ssl_stapling_verify on;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}

# 顺便把 HTTP 重定向到 HTTPS
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}
```

几个关键点：

1. `ssl_certificate` 必须是 fullchain（域名证书 + 中间证书），只放叶子证书会导致部分客户端报"证书链不完整"。
2. 私钥权限设严：`chmod 600 privkey.pem`，属主设成 root 或 nginx 用户。
3. 改完先测试再 reload

```bash
   nginx -t && nginx -s reload
```

1. 验证部署：用 `openssl s_client -connect example.com:443 -servername example.com` 看握手细节，或者直接上 [SSL Labs](https://www.ssllabs.com/ssltest/) 跑一下评分。

2. 多域名：如果一个 IP 上有多个 HTTPS 站，写多个 `server` 块，每个用自己的 `server_name` 和证书，nginx 通过 SNI 自动区分即可。

   

4. 访问时，服务器把公钥主动发给客户端,客户端不需要事先知道。

5. HTTPS 握手时公钥怎么传：

TLS 1.2简化流程:

```
客户端                                  服务器
  │                                       │
  │──────── ClientHello ─────────────────>│  我支持这些 TLS 版本和 cipher
  │                                       │
  │<─────── ServerHello + 证书 ───────────│  这是我的证书(里面有公钥)
  │                                       │
  │ 验证证书:                              │
  │ - 是不是受信任 CA 签的?                 │
  │ - 域名对不对?                          │
  │ - 有效期内吗?                          │
  │                                       │
  │──── 服务器用证书里公钥对应的私钥给握手参数签名，
         客户端用证书里的公钥验证签名，
           确认对方确实持有私钥 ────────────>│
  │                                       │
  │<──────── 握手完成,开始加密通信 ────────│
```

关键点:服务器证书(`ssl_certificate` 指向的 `fullchain.pem`)本身就包含了公钥。客户端连上来时,服务器把这张证书发过去,公钥就传给客户端了。

6. 那"公钥"到底放在哪里？

回到证书内容,服务器上你只需要保管两个东西:

| 文件                   | 含什么                                 | 谁能看                         |
| ---------------------- | -------------------------------------- | ------------------------------ |
| `fullchain.pem` (证书) | **公钥** + 域名信息 + CA 签名 + 证书链 | 公开,谁连服务器都会拿到        |
| `privkey.pem` (私钥)   | **私钥**                               | 绝密,只能放在服务器上,权限 600 |

公钥已经嵌在证书里了,你不需要再单独存一个公钥文件给客户端。

7. 客户端怎么"信任"这个公钥

这就是 CA(证书颁发机构)体系的作用:

1. 客户端(浏览器/操作系统)预装了一堆受信任的根 CA 证书(比如 Let's Encrypt、DigiCert、GlobalSign 的根证书)
2. 你的域名证书是被某个 CA(或它的中间 CA)签过名的
3. 客户端拿到你的证书后,用 CA 的公钥验证你的证书签名——签名对得上,就证明这张证书(以及里面的公钥)是真的属于这个域名的
4. 验证通过后,客户端就放心用证书里的公钥参与后续握手

所以信任链是:客户端预装的根 CA → 中间 CA → 你的域名证书(含公钥)。

8. 客户端不需要预先有你的公钥

普通用户访问 `https://example.com`:

- ❌ 不需要事先下载你的公钥
- ❌ 不需要安装你的证书
- ✅ 只要他的浏览器/系统信任签发你证书的 CA 就行

这就是 HTTPS 能给陌生人用的根本原因——信任根 CA,而不是信任每一个网站。

9. 什么时候才需要手动给客户端公钥/证书

只有这些特殊场景:

1. 自签名证书(没有 CA 签):客户端必须手动把你的证书加到信任库,否则会报"不安全"
2. 私有 CA:公司内网自己当 CA,得把这个内部根 CA 装到员工电脑上
3. mTLS(双向认证):服务器还要验证客户端的身份,这时客户端也得有自己的证书和私钥,服务器需要客户端 CA 证书来验证

普通公网 HTTPS(用 Let's Encrypt、阿里云、腾讯云这种正规 CA 签的证书)啥都不用提前给客户端,买完证书部署到 nginx,谁都能直接访问。



pem中存了公钥吗？我怎么用私钥计算公钥？

1. PEM 文件里真的物理存了公钥

一个 RSA 私钥 PEM 文件解码出来,是个 ASN.1 结构,叫 `RSAPrivateKey`(PKCS#1 标准),长这样:

```
RSAPrivateKey ::= SEQUENCE {
    version           INTEGER,       -- 版本号
    modulus           INTEGER,       -- n  ← 公钥组成 1
    publicExponent    INTEGER,       -- e  ← 公钥组成 2
    privateExponent   INTEGER,       -- d  ← 私钥核心
    prime1            INTEGER,       -- p
    prime2            INTEGER,       -- q
    exponent1         INTEGER,       -- d mod (p-1)
    exponent2         INTEGER,       -- d mod (q-1)
    coefficient       INTEGER        -- q^(-1) mod p
}
```

注意前两个字段 n 和 e——这就是公钥的全部内容。RSA 的公钥就是 `(n, e)` 这一对数,而它俩就大大方方地存在私钥文件的开头。

`openssl rsa -pubout` 做的事就是:把前两个字段 `(n, e)` 单独抠出来,重新打包成公钥格式输出。

```bash
openssl rsa -in KeyPair-flagship-prod-com.pem -text -noout
```

输出会是类似这样:

```
Private-Key: (2048 bit, 2 primes)
modulus:                    ← 这就是 n,公钥的一部分
    00:c1:23:45:67:...
publicExponent: 65537 (0x10001)   ← 这就是 e
privateExponent:            ← 这是 d,私钥的核心
    00:9a:bc:de:...
prime1:                     ← p
    ...
prime2:                     ← q
    ...
```

2. 数学上也根本不需要"存",能算

即使私钥文件只存了 `d, p, q`(私钥本质部分),也能算出公钥:

- `n = p × q`(两个大质数相乘)
- `e` 一般固定是 `65537`,而且就算不固定,也能从 d 和 p、q 反推

所以"从私钥导出公钥"在 RSA 里是天然成立的,只是 PEM 格式为了快、也为了通用,直接把公钥部分一起存了,省得每次用都要重算。

3. 反过来呢:能从公钥推出私钥吗?

不能。这就是 RSA 的安全基础:

- 公钥 = `(n, e)`
- 私钥 = `(d)`(加上 `p, q` 之类的辅助)
- 算出 `d` 需要知道 `p` 和 `q`
- 而 从 `n` 分解出 `p × q` 在 `n` 足够大(比如 2048 位)时,以现有算力是不可行的

这叫大整数分解难题,RSA 的安全性完全建立在这个数学难题之上。信息是不对称的:私钥包含公钥,反过来不行。

4. 一个常见场景验证

你手上有个私钥,想知道对应的证书/公钥是不是匹配:

```bash
# 从私钥算出公钥指纹
openssl rsa -in privkey.pem -pubout -outform der | openssl sha256

# 从证书取出公钥指纹  
openssl x509 -in cert.pem -pubkey -noout | \
  openssl pkey -pubin -outform der | openssl sha256
```

两个哈希一样,说明这张证书和这个私钥是配对的——这是部署 HTTPS 之前常做的"配对检查"。

验证：

```
ssh-keygen -y -f KeyPair-zeyqaq-com.pem
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDE+mDrHcoMnnNhSveG45ILfesnEUTL/RfcsmHIPGnhj9k+fLOEJ2asSUf0L+tjcwcSEzrakNznI0j8y3b9L8JCTit1uZCXrI/nzIHisrbArytWH6nJDrS757b4TEliPVYD01u2mDAba9bsF1vW9T2qK3Ibs2HjHVIo+3Q8MOnU7FgndPMsasSWTqhIZTWS1DrYgal6fbYfq/MplxNy+qmQ3NUS9dYKVoPpCwZGNqFoiDBGQprgOl/9D2EWl8WoOz1jTtjPebLrfZJMglkQpMT5UfEtEBvwui1ShtwPUJUwoeTOFyb31O7XQCICmBXnt0pKD2A6UkK8HD8ugYma4U6l

[root@ecs-zeyqaq eddie]# cat ~/.ssh/authorized_keys
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDE+mDrHcoMnnNhSveG45ILfesnEUTL/RfcsmHIPGnhj9k+fLOEJ2asSUf0L+tjcwcSEzrakNznI0j8y3b9L8JCTit1uZCXrI/nzIHisrbArytWH6nJDrS757b4TEliPVYD01u2mDAba9bsF1vW9T2qK3Ibs2HjHVIo+3Q8MOnU7FgndPMsasSWTqhIZTWS1DrYgal6fbYfq/MplxNy+qmQ3NUS9dYKVoPpCwZGNqFoiDBGQprgOl/9D2EWl8WoOz1jTtjPebLrfZJMglkQpMT5UfEtEBvwui1ShtwPUJUwoeTOFyb31O7XQCICmBXnt0pKD2A6UkK8HD8ugYma4U6l Generated-by-Nova
```



私钥 不是一个用密码加密保护的数据。它本身就是一组很大的随机数，那为啥我连接的时候，用明文密码，类似于“1234567”，也能登陆？

1. 那说明服务器**同时**开了密码登录

SSH 服务器(`sshd`)支持多种认证方式,可以同时启用。常见的两种是:

1. 公钥认证(PubkeyAuthentication):用 `.pem` 私钥登录,不输密码
2. 密码认证(PasswordAuthentication):用用户名 + 密码登录

两种是并列关系,不是二选一。服务器开了哪种,客户端就能用哪种登。你能用密码登录,说明这台机器的 `sshd` 配置里 `PasswordAuthentication yes`。

2. 这个密码是哪来的

不是从 .pem 里"算"出来的,它是系统用户的登录密码——和 Linux 普通用户登录系统的密码是同一个,存在服务器的 `/etc/shadow` 文件里(以哈希形式)。

可能的来源:

- 服务器管理员给你设的:有人执行过 `passwd ec2-user` 给账号设置了密码
- 创建用户时设的:`useradd` 时给的密码
- 云厂商默认密码:某些镜像(尤其是非 AWS 官方的)会预置密码
- 你自己之前设的:可能你自己 `passwd` 改过

总之这个 `1234567` 是有人主动设置到那个 Linux 账号上的密码,和 `.pem` 私钥毫无关系——两条独立的登录通道,只是恰好都通向同一个账户。****