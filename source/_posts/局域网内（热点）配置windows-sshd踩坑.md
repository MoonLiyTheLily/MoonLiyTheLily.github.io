---
title: 局域网内（热点）配置windows sshd踩坑
date: 2026-09-18 18:24:31
tags:
---

# 局域网内（热点）配置Windows sshd踩坑

配置环境：一台笔记本，Windows 11，配置了 Windows 可选功能内的`sshd`，作为主机（记为电脑A）。另一台笔记本，只安装了 ssh client，作为客户端（记为电脑B）。

## Windows的局域网防火墙配置和热点问题

Windows 的热点，在 Windows 11 似乎是用 WLAN Direct 来实现的，所以在设置-高级网络设置处，查看所有网络适配器，开启热点的时候会有一个 Wi-Fi Direct Virtual Adapter 的适配器。

而这个网络默认是**公用的**，实在是特别奇异。由于这个，不得不配置防火墙的规则。

### 弯路：关于试图ping通主机的尝试

本人先试图 `ping` 了主机但是不通，一开始以为是 ip 不对，还去查了 WLAN Direct 是不是有什么特殊的机制。因为看到 B 上 `ipconfig` 显示的网关就是 A 上显示的 A 的 ip，不过好像也没问题就是了。

后来我抓了个包（牛刀杀鸡.jpg），发现其实 B 的 ping 数据包发送过来 了，但是 A 没有做出回应。所以并不是 ip 不对。所以我查了一下为什么局域网 ping 不通，答案是需要去打开**文件和打印机共享**服务。在资源管理器，网络页面，最上面应该会出现一个小黄条，打开即可。

![img1-1](/images/img1-1.png)

打开之后，如果正常，应该在高级防火墙设置的出站、入站规则里都能看见文件和打印机共享的一个叫回显请求（ICMP）在公用网络是允许的。



### SSH

既然 ping 都这么严，ssh 就更不行了。需要专门去防火墙里配置 OpenSSH 的规则。通过 Windows 安全中心->防火墙->高级设置进入。然后在入站规则里找到 OpenSSH，右键属性->高级，勾选公用即可。

![img1-2](/images/img1-2.png)

*这个东西甚至默认不是按字母排序的*



## sshd的登陆配置

现在本地网络应该能看见SSH的连接了。如果没法登陆是正常的，因为还没修改 sshd 的配置文件。

首先，去 C:\ProgramData\ssh\sshd_config 修改，将

```
Match Group administrators
       AuthorizedKeysFile __PROGRAMDATA__/ssh/administrators_authorized_keys
```

注释掉。

这一步可能导致一些奇怪的问题，比如 sshd 莫名其妙无法启动了，报错108几还是106几，忘记了，只记得是1开头，此时只需要删除整个ssh文件夹然后重启电脑，再修改，应该就不会有问题了。

如果注释掉了还无法登录，可能是由于 Windows 默认的设置非常严，需要修改 C:\ProgramData\ssh\sshd_config，允许你的用户从 SSH 登录（在文件后追加`AllowUsers <your_username>`）。



## 关于免密登录

免密登录需要两个条件，第一个是自己生成了密钥，第二个是密钥的公钥在服务器的 authorized_keys 里。

生成公钥此处跳过，只提一些奇怪的坑。

首先，需要修改（服务器上）你自己文件夹下的 authorized_keys 的权限。

![img1-3](/images/img1-3.png)

右键属性->安全->高级->禁用继承->转换为显式权限。

其次，Windows Powershell 的`cat`似乎有一些奇怪的问题导致`cat>>authorized_keys`结果不正确，为了保险，建议是要么自己复制一个（也可能出错，原因未知），要么用 WSL 的`cat`。我因为只是局域网用用，只有一个电脑，就直接把我的 pub 文件复制上去改名叫`authorized_keys`了。

配置完成，现在应该能直接免密登录了。还可以通过内网穿透在外面登录。最后，建议把宿主机的 sshd 设置成开机启动。

```powershell
Set-Service -Name sshd -StartupType 'Automatic'
```

