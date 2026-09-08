很多时候我们需要用到镜像站，原因可能是我们在中国，也有可能是在一家有同步需求的公司。

非官方的APT源有着非常快的同步速度，也带来了夹带私货的风险，除了用户自发比对MD5 值/SHA256 值以外，APT官方应用了一种超时强制过期的机制，就类似各个网站的证书所有的那种。

这就避免了：即使我的证书是正确的，但是这是过期的系统/应用，存在漏洞。



先讲APT的信任链：

一个debian仓库是这样的：

```
http://mirror/debian/
├── dists/
│   └── bullseye/                    ← suite（发行版/套件）
│       ├── InRelease                ← 内联签名的元数据（信任根）
│       ├── Release                  ← 明文元数据
│       ├── Release.gpg              ← Release 的分离签名
│       ├── main/                    ← component（组件）
│       │   ├── binary-amd64/        ← architecture
│       │   │   ├── Packages         ← 包索引（明文）
│       │   │   ├── Packages.gz
│       │   │   └── Release          ← 小的 arch 级 Release（无签名）
│       │   └── source/Sources.gz
│       ├── contrib/
│       └── non-free/
└── pool/                            ← 实际的 .deb 文件都在这
    └── main/a/apt/apt_2.2.4_amd64.deb
```

`sources.list` 里写「deb http://mirror/debian bullseye main contrib」，APT 就自动拼出 「http://mirror/debian/dists/bullseye/InRelease」去拉。

release里有什么：

```
Origin: Debian
Label: Debian-Security
Suite: bullseye-security
Version: 11
Codename: bullseye
Date: Sun, 07 Sep 2026 20:15:32 UTC
Valid-Until: Sun, 14 Sep 2026 20:15:32 UTC
Acquire-By-Hash: yes
Architectures: amd64 arm64 i386 ...
Components: main contrib non-free
SHA256:
 3f1a...c9  1387422 main/binary-amd64/Packages
 8b7e...02   342915 main/binary-amd64/Packages.gz
 ...
```

Debian 官方 security 仓库的 `Valid-Until` 一般是 **7 天**（普通 archive 也是 7 天，Ubuntu 类似）。所以镜像必须至少每周同步一次，否则必过期。

信任链是如何串联的：

```
本地 keyring（/etc/apt/trusted.gpg.d/, /usr/share/keyrings/, signed-by=）
        │  用公钥验证 GPG 签名
        ▼
InRelease / Release+Release.gpg      ← 唯一的签名点
        │  用文件内的 SHA256 校验
        ▼
Packages / Packages.gz / Sources     ← 包索引，本身不签名
        │  索引里每个包条目带 SHA256 + Filename
        ▼
pool/main/a/apt/apt_2.2.4_amd64.deb
```

1. **只有 Release 被签名**，索引和 deb 包都不签。所以一旦 Release 的签名或时效校验失败，APT 就不敢信任下游任何东西 —— 这就是为什么"只是过期了 2 小时"也会让整个源直接失效。
2. APT 的标准仓库验证流程不要求逐个验证 `.deb` 的独立签名。因为它的哈希写在 Packages 里，Packages 的哈希写在 Release 里，Release 有签名。任何一环被篡改都会断链。



再讲为什么要有Valid-Until：

防的是**重放/降级攻击（replay / freeze attack）**：

> 攻击者（或恶意镜像）截断你的更新，一直给你一份**旧的但签名完全合法**的 Release。你以为系统是最新的，实际上停留在某个已知有 RCE 的版本上，且 APT 完全察觉不到——因为签名是真的。

`Valid-Until` 就是给"合法但过时"的元数据加一个自然死亡时间，把这类攻击窗口压到几天内。

代价就是：**镜像同步一停，源就必然过期**。



如果你apt-get update的时候出现了「Release file ... is expired」这样的报错，说明客户端拿到的那一份 Release 元数据已经超过 Valid-Until，那可以关注一下：

Debian 11 的 LTS（长期支持）官方生命周期在 2026年8月31日 正式结束。官方在停止支持后，最后一次生成并签名的 `Release` 文件大约是在 8月底或 9月初。 正如前文所说，`Valid-Until` 的有效期是 7天。因此，`8月31日（最后更新） + 7天缓冲期`，恰好导致了全球所有还在使用 Debian 11 Security 源的机器在 2026年9月7日～9月8日 左右集中爆发 `Release file is expired` 报错。

所有使用以上配置debian-security软件源的容器、镜像、主机等，在进行apt update/install时会出现失败，无法进行包的安装与更新。

短期解决方案：按源关闭check-valid-until=no，镜像站修复后恢复—— 保留源但绕过过期校验。

```
将security源配置按照如下修改：
deb http://security.debian.org/debian-security bullseye-security main contrib non-free
```

长期解决方案：更新基础镜像到debian12

并不那么好的解决方案：如果因业务原因确实无法立即升级到 Debian 12，除了绕过过期校验，更稳定的做法是将源替换为 Debian 归档镜像站（Archive），这些源专门用于存放已停止维护的版本。使用 archive 源时，APT 通常会自动处理或忽略归档仓库的过期时间。

```
deb http://archive.debian.org/debian/ bullseye main contrib non-free
deb http://archive.debian.org/debian-security/ bullseye-security main contrib non-free
```

