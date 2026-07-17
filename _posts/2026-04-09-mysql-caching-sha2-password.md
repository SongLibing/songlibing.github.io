---
title: 深入理解 MySQL caching_sha2_password 认证机制
date: 2026-04-09 10:00:00 +0800
categories: [数据库, MySQL]
tags: [MySQL, 认证, 安全, caching_sha2_password]
toc: true
---

## 密码认证的基本要素
基于密码的身份认证包括了两个部分：
1. 服务器端认证信息的存储
2. 密码的认证过程

基于密码的身份认证有一个原则：`仅使用者知道密码`。密码不能被存储在认证服务器中，在认证过程中也不能通过网络明文传输。因为存储的信息可能被窃取或者滥用，网络可能被监听。这些安全隐患都可能导致密码的泄露。

MySQL中常用的 `mysql_native_password` 和 `caching_sha2_password` 都遵循上述的要求。mysql_natvie_password 是MySQL-8.0之前最主要的认证方式。从`MySQL-5.7.23`开始，MySQL引入了新的认证方式 `caching_sha2_password`，并在`MySQL-8.0`将其设置为默认的认证方式。`mysql_native_password`则在`MySQL-8.4`被默认禁用，并且从`MySQL-9.0.0`之后被移除了。关于 mysql_native_password 的实现逻辑，可以参考《MySQL原生密码认证》一文。


## 为什么引入 caching_sha2_password 

`mysql_native_password` 诞生于 2000 年代初，使用 `SHA1(SHA1(密码))` 做存储和认证。在当时的计算能力和安全威胁模型下，这个方案是可靠的。随着时间推移，情况已经发生改变。GPU 和专用硬件使得暴力破解 SHA1 的成本大幅降低；预计算彩虹表在无盐哈希面前效率极高；2017 年 Google 和 CWI Amsterdam 完成了对 SHA1 的首次实际碰撞攻击（[SHAttered](https://shattered.io/)）。此后 NIST 正式建议弃用 SHA1（[NIST SP 800-131A Rev.2](https://csrc.nist.gov/pubs/sp/800/131/a/r2/final)）。在这样的背景下，MySQL-5.7 引入了 `caching_sha2_password`，来应对新的安全挑战，满足企业的合规需求。


## 认证信息的存储

### mysql_native_password 的存储格式

在《MySQL原生密码认证》中我们介绍过，`mysql_native_password` 在 `mysql.user` 表中存储的是两次 SHA1 的哈希值：

```
*0D3CED9BEC10A777AEC23CCC353A8C08A633045E
```

其计算方式为：

```
stage2hash = SHA1(SHA1(密码))
```

这个哈希值只有 20 字节（十六进制 40 字节），而且没有加盐，也就是说`相同的密码产生的哈希值总是一样的`。这就给暴力破解和彩虹表破解创造了便利条件。

### caching_sha2_password 的存储格式

`caching_sha2_password` 在 `mysql.user` 表的 `authentication_string` 字段中存储的格式如下：

```
$A$005$<20字节的salt><43字节的digest>
```

我们来看看每个部分的含义：
- `$`：分隔符，每一部分都是以`$`开始
- `$A`：摘要类型，A 表示 SHA256
- `$005`：哈希计算轮次的十六进制表示，实际次数 = 0x005 × 1000 = 5000 轮
- `$<salt><digest>`: 20 字节随机生成的salt(随机数)和43 字节Base64 编码的 SHA256 摘要

哈希计算轮次通过系统变量 `caching_sha2_password_digest_rounds` 配置，默认 5000 轮，最大可配置到 0xFFF × 1000 = 4,095,000 轮。这个参数是`只读的`，启动后不能动态修改。不同用户可以有不同的轮数，因为轮数编码在各自的 `authentication_string`字段中。修改该参数后，只有`新设置密码的用户`才会使用新的轮数，已有用户不受影响。密码最大长度为 256 字节。

相比 `mysql_native_password`，主要增强了两个方面：
- 加盐：每个用户使用 20 字节随机盐值，相同密码产生不同哈希值，攻击者必须针对每个用户独立计算
- 多轮哈希：默认 5000 轮 SHA256，单次破解的计算量远高于 `mysql_native_password` 的 2 轮 SHA1

## 两段式认证
多轮加盐哈希增加了破解的难度，但同时也带来了认证效率的下降。5000 轮哈希的计算太慢了，如果每次认证都需要计算哈希，会严重的降低认证的效率。

于是 `caching_sha2_password` 做了一个两阶段认证的设计：
- 快速认证(fast_auth)
- 完整认证(full_auth)

`首次认证`必须走完整认证（传输密码 → 多轮哈希验证 → 缓存快速摘要）。首次认证后服务器会缓存 `SHA256(SHA256(密码))` 到内存。`后续认证`走快速认证（scramble 验证，类似 `mysql_native_password`）

`这也是 "caching" 这个名字的由来——通过缓存快速哈希，将昂贵的多轮哈希验证转换为轻量级的 scramble 验证。`

之所以称作两阶段认证，是因为认证的过程默认使用的是快速认证。客户端会先执行快速认证的过程，如果认证失败，则会进行完整认证。

## 快速认证

### 1. 服务器发送 scramble 到客户端

服务器生成一个 20 字节的随机字符串（scramble），发送给客户端。这个随机字符串和 `mysql_native_password` 中的作用一样，用于生成一次性的加密秘钥。

### 2. 客户端生成并发送密文

客户端收到 scramble 后，使用 SHA256 算法生成 32 字节的密文发送给服务器：

```
digest_stage1 = SHA256(密码)
digest_stage2 = SHA256(digest_stage1)
key = SHA256(digest_stage2 + scramble)
密文 = XOR(digest_stage1, key)
```

这个过程和 `mysql_native_password` 的逻辑完全一致，区别只是将 SHA1 替换成了 SHA256，密文长度从 20 字节变成了 32 字节。
- 对密码进行两轮 SHA256 哈希 (digest_stage2) 后混合 scramble 产生一个加密密钥。
- 通过 XOR 的方式对密码的 SHA256 哈希(digest_stage1) 进行加密后发送给服务端。

### 3. 服务器验证密文

快速认证的前提是服务器的`内存缓存中已经存在该用户的快速认证哈希`。

服务器从内存缓存中取出该用户的快速摘要（即 `digest_stage2 SHA256(SHA256(密码))`），然后验证：

```
key = SHA256(digest_stage2 + scramble)
digest_stage1 = XOR(密文, key)
验证: SHA256(digest_stage1) == 快速摘要 ?
```
- 服务端通过缓存的 digest_stage2 哈希，生成和客户端相同的key
- 然后用这个 key 解密出 digest_stage1
- 最后用 SHA256(digest_stage1) 和自己缓存的 digest_stage2 哈希比对是否相同。
- 如果成功则发送`快速认证成功`的消息给客户端，否则发送完整认证的请求给客户端。

### 协议消息定义
除了快速认证成功消息，后面还有两个特殊的消息。他们都是只有`1`个字节的消息定义如下。
![](https://mmbiz.qpic.cn/mmbiz_png/Jg6JM4XTDce7gD4RcDd12wicqgkhBV52Eibh90KU7MdEvR0qokWIdKibG37nbWj9qiaJW9eqLTT0kjSfPAiaP8ZnFH0MMxtWcJ4fn4rICpZ3zgoA/640?wx_fmt=png&from=appmsg)

## 完整认证

当缓存中没有该用户的条目或快速认证验证失败时，服务器发送`要求客户端进行完整认证`。

### 客户端发送密码

完整认证需要客户端将用户的`原始密码`发送给服务端，因此需要有一个安全的传输通道。根据连接类型有两种处理方式：

- 安全连接（TLS / Unix Socket）：通道本身已加密或者是本地socket连接，直接发送明文密码。
- 非安全连接：需要使用服务端的 RSA 公钥对用户密码进行加密发送。在加密前，要先将用户密码和 scramble 做 XOR 运算。由于 scramble 是一次性的随机数，可以防止重放攻击。
```
xor_password = XOR(用户密码, scramble)
密文 = RSA_encrypt(xor_password)
```

如果客户端没有预配置 RSA 公钥，可向服务器发送公钥请求。服务器收到请求后，则返回 PEM 格式的公钥。客户端相关的连接参数：

- `--server-public-key-path`：预先指定服务器 RSA 公钥文件路径（推荐，避免中间人攻击）
- `--get-server-public-key`：允许客户端从服务器获取公钥（方便但存在安全风险）

如果非 TLS 连接且以上两个参数都未设置，客户端会报错`Authentication requires secure connection`。这是 DBA 从 MySQL-5.7 升级到 MySQL-8.0 后最常遇到的连接问题。

### 服务器验证密码

服务器收到密码后（RSA 加密的先解密还原），根据 `authentication_string`字段中的盐值(salt)和迭代次数，对明文密码计算 SHA256-crypt 多轮哈希，然后与user表中记录的摘要比对。

认证成功后，服务器生成快速摘要 `SHA256(SHA256(密码))` 并`写入内存缓存`，后续该用户的连接就可以走快速认证了。

完整的认证流程图如下所示:
![](https://mmbiz.qpic.cn/sz_mmbiz_png/Jg6JM4XTDcdzWmKuFp3jv6DE8YKDu0JlicY8YIBZEOBAoeI0hq1xyNf93Qw4gU9vSdapQ9wWE8KkAZFRYQrKHZseia487ibKFJZCBvibpTItLvw/640?wx_fmt=png&from=appmsg)

参考代码(MySQL-8.0)：`sql/auth/sha2_password.cc` 中的 `Caching_sha2_password::authenticate()`

## 缓存管理

### 缓存结构

缓存使用哈希表实现，key 是"用户名+主机名"，value 是该用户的快速摘要。读写锁保护并发访问：快速认证查找缓存时加读锁，完整认证成功写入缓存时加写锁。写锁只在写入缓存的瞬间持有，5000 轮哈希计算在锁外完成，不会阻塞其他连接的快速认证。

### 缓存失效

caching_sha2_password 实现了一个审计 plugin（`sha2_cache_cleaner`），通过监听认证相关的审计事件来管理认证缓存。之所以使用 audit plugin 而不是直接在 ACL 子系统中调用缓存清理接口，是为了保持松耦合——认证插件不需要侵入 ACL 模块的代码，只需监听事件即可响应用户变更。具体的失效规则：

- `FLUSH PRIVILEGES`：清空所有用户的缓存，之后所有用户需重新走完整认证
- `DROP USER`：删除该用户的缓存条目
- `RENAME USER`：删除旧用户名的缓存条目
- `ALTER USER`（修改密码）：删除该用户的缓存条目

参考代码：`sql/auth/sha2_password.cc` 中的 `sha2_cache_cleaner_notify()`

### 双密码支持

MySQL-8.0 支持 `ALTER USER ... RETAIN CURRENT PASSWORD`，允许一个用户同时拥有主密码和备用密码（旧密码）。

`caching_sha2_password` 对双密码的支持体现在：

- 存储层：`authentication_string`（主密码）和 `additional_auth_string`（备用密码）各自独立存储
- 内存缓存：每个用户的缓存条目包含两个槽位，分别存储主密码和备用密码的快速摘要
- 认证顺序：先验证主密码，失败后再验证备用密码。快速认证和完整认证都遵循这个顺序
- 审计标记：如果用户通过备用密码认证成功，服务器会记录一条日志提示管理员

## 与 mysql_native_password 的兼容

MySQL-5.7 升级到 MySQL-8.0 后，已有的 `mysql_native_password` 用户仍然使用原来的认证插件不受影响。如果需要将用户切换到 `caching_sha2_password`，需要通过`ALTER USER` 进行调整。创建新用户时，默认使用`caching_sha2_password`。


## 安全性分析

### 与 mysql_native_password 的对比
![](https://mmbiz.qpic.cn/mmbiz_png/Jg6JM4XTDcedp7kxckDDsFoQGJzicbPoSiap6Mdia6lLFes8ibj9RcXkDv51PPUePl2pH6fXRa1kR9bjDmQgeIt1xcKjibretDrC78r57ibkOo3A4/640?wx_fmt=png&from=appmsg)

### 服务器上的哈希值被盗

由于使用了`加盐 + 多轮哈希`，即使 `authentication_string` 泄露，攻击者也需要针对每个用户独立计算 5000 轮哈希才能尝试破解。而 `mysql_native_password` 未使用盐值，相同密码的哈希值相同，攻击者可以用一次计算结果匹配所有使用相同密码的用户。

### 内存缓存被盗

如果攻击者能读取服务器进程内存，获取到缓存的快速摘要 SHA256(SHA256(密码))，攻击者`不能`直接用它通过快速认证。因为认证需要的是 `SHA256(密码)`，而从 `SHA256(SHA256(密码))` 无法反推出 `SHA256(密码)`。但攻击者可以用它做离线暴力破解或者彩虹表破解。相比从磁盘上的 `authentication_string` 破解（每次尝试 5000 轮）代价低得多。这和偷到 `mysql_native_password` 中 `mysql.user` 存储的 `SHA1(SHA1(密码))` 的情况是一样的。


### 网络被监听

- 快速认证：和 `mysql_native_password` 安全性相同。每次 scramble 不同，无法重放或反推。

- 完整认证：需要传输密码，因此必须通过 TLS 或 RSA 加密保护。这里有一个需要注意的风险：如果客户端从服务器获取 RSA 公钥（而非预先配置），在中间人攻击场景下，攻击者可以替换为自己的公钥。`建议在不安全的网络环境中预先配置 RSA 公钥，或直接使用 TLS 连接`。

## 总结

从 `mysql_native_password` 到 `caching_sha2_password`，是 MySQL 密码认证体系随安全威胁演进而自然升级的结果。`mysql_native_password` 在它的时代是可靠的设计；`caching_sha2_password` 则在继承其 scramble 验证思想的基础上，针对现代安全威胁引入了加盐和多轮哈希，并通过缓存机制平衡了安全性和性能：

- 存储层：加盐 + 5000 轮 SHA256-crypt，大幅提升单次破解的计算量
- 认证层：缓存 SHA256(SHA256(密码))，继承 `mysql_native_password` 的 scramble 验证思想
- `首次连接`走完整认证（慢但安全），`后续连接`走快速认证（快且安全）

使用建议：
- 优先使用 TLS 加密连接
- 非 TLS 环境下，客户端预先配置 RSA 公钥（`--server-public-key-path`）
- 了解 `FLUSH PRIVILEGES` 和服务器重启会清空缓存，可能导致短暂 CPU 尖刺
- `caching_sha2_password.digest_rounds` 可调整存储哈希轮数，轮数越高越安全但完整认证越慢

