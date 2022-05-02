# Windows Git 代理配置


<!--more-->
## 1 http/https 代理
查看代理
```bash
git config --global --get http.proxy
git config --global --get https.proxy
```
配置代理
```bash
git config --global http.proxy http://ip:port
git config --global https.proxy https://ip:port
```
取消配置
```bash
git config --global --unset http.proxy
git config --global --unset https.proxy
```
{{< admonition note >}}
以上配置只适用于 http/https链接
{{< /admonition >}}
## 2 ssh 代理
ssh代理通过修改ssh配置文件实现, 查看`~/.ssh/config`, 如果没有则创建, 添加如下配置
```
Host github.com
  Hostname ssh.github.com
  User git
  Port 443
  # 注意修改路径为你的路径
  IdentityFile "C:\Users\Reload\.ssh\id_rsa"
  ProxyCommand  connect.exe -H 127.0.0.1:7890 -a none %h %p
  TCPKeepAlive yes
```
**Host**指定了下列配置针对的主机名称规则, 可以使用通配符`*`(0-n字符)和`?`(1个字符)
{{< admonition example >}}
github.com 可以匹配 `ssh username@github.com`, 不能匹配`ssh username@ssh.github.com`

*github.com 则上述两种都可以匹配
{{< /admonition >}}
**Hostname**指定了要ssh连接的真实地址, 也就是将匹配的Host替换为Hostname
{{< admonition example >}}
应用如下配置
```
Host myserver
    Hostname github.com
```
效果如下
{{<image src="ssh_hostname_example.png" caption="ssh连接效果">}}
{{< /admonition >}}
**User**指定了要连接的用户名, **Port**指定了要连接的端口

**IdentityFile** 是对应私钥的存放路径

**ProxyCommand** 是在连接前执行的指令, 这里使用connect工具(windows 版本 git 自带)

connect的主要参数  `[-H proxy-server[:port]] [-S [user@]socks-server[:port]]`

所以 **-H** 对应http代理, **-S** 对应socks5代理.
这里的 **-a none** 是 NO-AUTH 模式的含义

**%h** **%p** 是ssh配置文件中默认的占位符, 分别代表替换后的hostname和port

{{<admonition note>}}
在网上可以看到有两种配置
```
Host github.com
  Hostname ssh.github.com
  User git
  Port 443
```
另一种是
```
Host github.com
  Hostname github.com
  User git
  Port 22
```
这两者的区别就是hostname和port不同, 都是有效的. 有些防火墙禁止了22端口的数据, 此时可以使用443端口[^1](目前使用的机场就ban了22端口, 之前使用的下面的配置, 最近发现不能用了, 才有了这篇文章)
[^1]: [using ssh over the https port](https://docs.github.com/en/authentication/troubleshooting-ssh/using-ssh-over-the-https-port)
{{</admonition>}}

{{<admonition note>}}
**ProxyCommand** 中的 connect是在git环境里的, 所以在git push/clone时是可以正确执行的, 但是如果单独执行就会缺失相应的环境变量. 因此, 即使配置正确, `ssh -T git@github.com`的测试也不能正确执行, 除非将connect加入windows path中
{{</admonition>}}

{{<admonition tip>}}
可以使用 **-v** 参数查看ssh的调试信息, 如`ssh -v -T git@github.com`
{{</admonition>}}
