---
date: "2017-12-16T21:31:00+08:00"
title: "安全配置ssh服务"
menu: "main"
Categories: ["Development", "SSH", "linux"]
Tags: ["Development", "Bash"]
Description: "配置SSH服务，记录方法方便以后查看"
---

## 基础 ssh 服务端配置

```text
Port 10837
Protocol 2
PermitEmptyPasswords no
PermitRootLogin no
AllowUsers xxx
DenyUsers www
ChallengeResponseAuthentication no
GSSAPIAuthentication no
HostbasedAuthentication no
KbdInteractiveAuthentication no
KerberosAuthentication no
PasswordAuthentication yes
PubkeyAuthentication yes
RhostsRSAAuthentication no
RSAAuthentication no
UsePAM yes
```

- 修改端口为非标准端口[服务使用端口列表](https://zh.wikipedia.org/wiki/TCP/UDP端口列表)
  - `firewall-cmd --zone=public --add-port=10837/tcp --permanent`
  - 需要修改防火墙规则
- 使用第二代通信协定
- 不容许空白密码
- 禁止root使用SSH登入
- 启用公钥验证
  - ssh-keygen [-t dsa | ecdsa | ed25519 | rsa | rsa1] -b 4096 
  - ed25519无法指定长度
  - ssh-copy-id [-i [identity_file]] [-p port] [user@]hostname
  - 或者直接拷贝公钥到相应账号的 ~/.ssh/authorized_keys 文件中。
- 限制可以登录的用户为xxx  (`AllowGroups`)
- 拒绝登陆的用户为www      (`DenyGroups`)
- 停用不需要的验证方法
- 在SELinux环境下可以保留使用PAM，避免停用PAM而没有加入适当的SELinux规则，会引致不可预知的结果。
- systemctl restart sshd.service, 重启 centos7 ssh 服务应用配置。

## 进阶 ssh 服务端配置

> 基础部分配置容易理解且对用户体验较少，进阶部分比较复杂且不宜理解需要管理员酌情考虑

###  密钥交换算法

> ssh 使用的算法有上述8种方法，推荐使用的算法为1和5两项算法，其它的均存在安全隐患。1算法的效率较高，5算法兼容性较好，联合使用较好。

  1. curve25519-sha256: ECDH over Curve25519 with SHA2
  2. iffie-hellman-group1-sha1: 1024 bit DH with SHA1
  3. diffie-hellman-group14-sha1: 2048 bit DH with SHA1
  4. diffie-hellman-group-exchange-sha1: Custom DH with SHA1
  5. diffie-hellman-group-exchange-sha256: Custom DH with SHA2
  6. ecdh-sha2-nistp256: ECDH over NIST P-256 with SHA2
  7. ecdh-sha2-nistp384: ECDH over NIST P-384 with SHA2
  8. ecdh-sha2-nistp521: ECDH over NIST P-521 with SHA2

```text
KexAlgorithms curve25519-sha256@libssh.org,diffie-hellman-group-exchange-sha256
```
> 第五项DH算法使用时 ssh 会从 /etc/ssh/moduli 随机选取长度合适的模数。由于长度过短或主机不是由自己安装（即仅克隆安装）的系统存在风险。

#### 自行安装的系统

> 系统安装时自动随机产生模数，我们仅筛选长度超过2048的模数即可。

```bash
awk '$5 >= 2047' /etc/ssh/moduli > "${HOME}/moduli"
wc -l "${HOME}/moduli"
mv "${HOME}/moduli" /etc/ssh/moduli
```

此处的模数必须有，即 `wc - l "${HOME}/moduli"` 结果大于0。

#### 非自主安装或克隆安装系统

> 此时系统内的 moduli 为多个系统重复，没有保密性科研，需重新制作。

```bash
cd ~
ssh-keygen -G moduli.all -b 4096
ssh-keygen -T moduli.safe -f moduli.all
mv moduli.safe /etc/ssh/moduli
rm moduli.all
```

以上的步骤非常耗时，可以考虑制作成脚本后台运行生产后，再进行操作。

### 验证服务器

> 服务器连接时如何验证是否是真实的，而非黑客冒牌的服务器呢。使用公钥加密验证进行验证。

- DSA with SHA1
- ECDSA with SHA256, SHA384 or SHA512，視乎密鑰的長度
- Ed25519 with SHA512
- RSA with SHA1

> ssh 服务支持上述四种公钥加密，dsa 算法只支持 1024 bit 密钥，安全性相比较低。ecdsa 由 NIST 提供，可信度较低。dsa 算法比较依赖高质素的随机数产生器。所以通常选择3、4两种算法，算法4虽然是RSA配合SHA使用，于安全性无害，因为真正计算散列值的函数是SHA2.

```text
HostKey /etc/ssh/ssh_host_ed25519_key
HostKey /etc/ssh/ssh_host_rsa_key
```

此时应配置如上加入到 `/etc/ssh/sshd_config` ，有其他的应当删除或注释。
此处使用的公钥也应该自行产生构造，保证安全

```bash
cd /etc/ssh
rm ssh_host_*key*
ssh-keygen -t ed25519 -f ssh_host_ed25519_key > /dev/null
ssh-keygen -t rsa -b 4096 -f ssh_host_rsa_key > /dev/null
```

### 验证用户

> 进行双因素验证，以保证本地客户端安全。

```text
PubkeyAuthentication yes
PasswordAuthentication yes
AuthenticationMethods publickey,password
```

### 限制可登陆用户

```bash
groupadd ssh-user
usermod -a -G ssh-user
```

```text
AllowGroups ssh-user
```
加入到 `/etc/ssh/sshd_config` 中,如此可方便授权使用ssh登陆。

### 加密算法

> 选择可靠的对称加密法

  - 3des-cbc
  - aes128-cbc
  - aes192-cbc
  - aes256-cbc
  - aes128-ctr
  - aes192-ctr
  - aes256-ctr
  - aes128-gcm@openssh.com
  - aes256-gcm@openssh.com
  - arcfour
  - arcfour128
  - arcfour256
  - blowfish-cbc
  - cast128-cbc
  - chacha20-poly1305@openssh.com

我们最后的加密法名单是15，9，8，另外加上7，6，5三个具备ctr工作模式的加密法作为后备，如果你全部SSH客户端都支持15，9，8三种加密法，便无需7，6，5了。

```text
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr
```
### 信息鉴别码

```text
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,hmac-ripemd160-etm@openssh.com,umac-128-etm@openssh.com,hmac-sha2-512,hmac-sha2-256,hmac-ripemd160,umac-128@openssh.com
```

  - hmac-md5
  - hmac-md5-96
  - hmac-ripemd160
  - hmac-sha1
  - hmac-sha1-96
  - hmac-sha2-256
  - hmac-sha2-512
  - umac-64
  - umac-128
  - hmac-md5-etm@openssh.com
  - hmac-md5-96-etm@openssh.com
  - hmac-ripemd160-etm@openssh.com
  - hmac-sha1-etm@openssh.com
  - hmac-sha1-96-etm@openssh.com
  - hmac-sha2-256-etm@openssh.com
  - hmac-sha2-512-etm@openssh.com
  - umac-64-etm@openssh.com
  - umac-128-etm@openssh.com

> 我們不會考慮 MD5 和 SHA1 散列函式。雖然經過 HMAC 包裝後，我們不能再對 MD5 和 SHA1 進行衝突攻擊 (collision attack)，消除了這兩個函式最致命的毛病，但是何必還糾纏於這些破破爛爛的函式？儘快把這些舊東西扔掉吧。
一定要優先使用 EtM，其他算法只能在客戶端不支援 EtM 時才使用。
散列函式的輸出，必須最少 128 bit，所以刪除了 umac-64。
加密鑰匙的長度必須最少 128 bit，這跟我們對加密算法的要求是一致的，幸運地所有 MAC 算法都符合這一項要求。

### 自动离线

```text
ClientAliveInterval 60
ClientAliveCountMax 5
LoginGraceTime 120
```

1. 指示 SSH 當客戶端連續 60 秒沒有送來任何訊息，便查詢一下它是否仍在。
2. 指示 SSH 連續 5 次收不到客戶端的回覆便自行斷線。
3. 指示 SSH 倘若用戶在 120 秒內不能成功登入便自動斷線。防止暴力破解。

### 登陆检测

```text
MaxAuthTries 6
MaxStartups 100
MaxStartups 10：30：100
```

指示SSH倘若用户连续6次不能登入便自动断线
指示SSH最多容许100个未登入的联机
指示SSH当未登入的联机达到10个的时候，新的联机便会有30%的机率被中断，这个机率随着未登入的联机数量上升而线性递增，当数量达到100个的时候，便完全拒绝新的联机。可适当调整最后一个数字，防止DDos时不容易被导致动弹不得。


### 其他安全配置

```text
HostbasedAuthentication no
ListenAddress  0.0.0.0
UsePrivilegeSeparation sandbox
Compression delayed
TCPKeepAlive no
UseLogin no
AcceptEnv LANG LC_*
```

1. 关闭预设开关。
2. 比较难以控制，能用则用。将ip地址改为相应会使用的ip。
3. 告诉SSH服务器每一个步骤使用不同的进程（process）执行，每个进程只具备必须的权限。防止过多破坏。
4. 将压缩功能延后到成功验证的用户后。
5. 基于它的安全问题，不应该开启这个选项
6. 关闭使用 linux login 指令来验证用户
7. 只接受文字语言相关的环境变量设定及以 LC_ 为首的变量名称

## 其他辅助工具

> 已超过本文谈论的范围。

- ufw – 防火墙设定工具，预防大量的登入请求
- ban2fail – 监控SSH日志找出可疑的黑客活动

## 案例配置

> 小明管理一台 centos 7 服务器，且使用selinux安全策略，要求安全性高，有公司30左右小伙伴一起登陆使用，人员调动频繁。安装系统时使用默认模板使用。小明使用账号 xiaoming ，服务器 ssh 端口 10837 。开启双因素认证登陆。例如有jonin用户登陆。要求兼容性较好。

`/etc/ssh/sshd_config` 配置

```text
Port 10837
Protocol 2
PermitEmptyPasswords no
PermitRootLogin no
AllowUsers xiaoming
AllowGroups ssh-user
ChallengeResponseAuthentication no
GSSAPIAuthentication no
HostbasedAuthentication no
KbdInteractiveAuthentication no
KerberosAuthentication no
RhostsRSAAuthentication no
RSAAuthentication no
UsePAM yes

KexAlgorithms curve25519-sha256@libssh.org,diffie-hellman-group-exchange-sha256
HostKey /etc/ssh/ssh_host_ed25519_key
HostKey /etc/ssh/ssh_host_rsa_key


PubkeyAuthentication yes
PasswordAuthentication yes
AuthenticationMethods publickey,password

Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,hmac-ripemd160-etm@openssh.com,umac-128-etm@openssh.com,hmac-sha2-512,hmac-sha2-256,hmac-ripemd160,umac-128@openssh.com

ClientAliveInterval 60
ClientAliveCountMax 5
LoginGraceTime 120
MaxAuthTries 6
MaxStartups 10:30:100

HostbasedAuthentication no
ListenAddress 0.0.0.0
UsePrivilegeSeparation sandbox
Compression delayed
TCPKeepAlive no
UseLogin no
AcceptEnv LANG LC_*
```

相应的调整 

```bash
firewall-cmd --zone=public --add-port=10837/tcp --permanent

cd ~
ssh-keygen -G moduli.all -b 4096
ssh-keygen -T moduli.safe -f moduli.all
mv moduli.safe /etc/ssh/moduli
rm moduli.all

cd /etc/ssh
rm ssh_host_*key*
ssh-keygen -t ed25519 -f ssh_host_ed25519_key > /dev/null
ssh-keygen -t rsa -b 4096 -f ssh_host_rsa_key > /dev/null

groupadd ssh-users
usermod -a -G ssh-users jonin

systemctl restart sshd.service
```

## 参考资料

1. [CentOS 7 ssh 安全配置一](https://www.hkpug.net/2015/12/16/centos-7-下安全配置-ssh-一/)
2. [CentOS 7 ssh 安全配置二](https://www.hkpug.net/2016/08/08/centos-7-下安全配置-ssh-二/)
