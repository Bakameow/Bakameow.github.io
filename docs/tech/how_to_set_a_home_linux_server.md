# 家庭Linux服务器捣鼓小记

## 前言
因为Baka最近爱上了Linux开发环境，故意图利用旧电脑搭建一个服务器。

## 准备工作
首先，我们需要一台闲置的电脑。

使用[Rufus](https://rufus.ie/zh/)和下载的Ubuntu24.04桌面版iso（不装Server版是因为~~不会通过命令行配置有线网卡~~)制作U盘启动盘，为旧电脑安装Ubuntu操作系统。

## IPv6公网访问

众所周知，家庭中的网络环境基本都使用了NAT技术，家中的网络设备获得的IP地址都是私网IP，当我们出门在外时无法直接连接家中的网络设备。

我曾经使用ToDesk、向日葵等工具，配置过Windows平台的Wake-on-LAN，实现了远程桌面控制。但我目前进行开发时只需要建立ssh连接，通过CLI的neovim即可完成开发，无需GUI。故需要一个更优雅的方案，完成内网穿透。

经过一番调研，我最后在Zerotier（虚拟局域网）和公网IPv6中选择了后者。

首先在GUI界面为Ubuntu服务器配置IPv6地址。如果服务器无法获得IPv6地址，可能需要进入路由器或光猫开启相关设置。

使用服务器浏览器连接[IPv6 测试](https://test-ipv6.com)对服务器的IPv6状态进行测试。如果获得了10分的好成绩，那你就吴迪了:P

在Ubuntu服务器中安装ssh服务
```bash
sudo apt install openssh-server
```

使用其他电脑设备连接手机的热点（模拟外网的访问行为），尝试ssh连接家中的服务器
```bash
# -6 指定使用IPv6地址建立连接
ssh -6 baka@<IPV6 ADDRESS>
```

如果无法建立ssh会话，但在IPv6测试中正常，可能是家中的路由器或光猫开启了IPv6防火墙，需要对其进行关闭（移动光猫更改设置需要登录超级用户，我的中国移动光猫型号TEWA272G，测试了网络上获取超级密码的许多方法后都以失败告终，电话联系移动宽带业务师傅被拒绝，最后上闲鱼买了XD）

实现公网环境下使用IPv6地址对家中服务器的访问后，下面配置DDNS。

## DDNS
在[Namesilo](https://www.namesilo.com/)（支持支付宝）或直接在阿里云上购买域名，并将域名托管到Cloudflare。

参考[GitHub - jeessy2/ddns-go](https://github.com/jeessy2/ddns-go)在服务器中部署ddns-go的docker容器
```bash
docker run -d --name ddns-go --restart=always --net=host -v /opt/ddns-go:/root jeessy/ddns-go
```

ddns-go将在9876绑定一个网页控制前端。可以在ubuntu服务器中使用浏览器访问`localhost:9876`，设置用户名和密码后，将Cloudflare的API key拷贝入ddns-go的控制面板。并在控制面板设置对应的DDNS解析地址，ddns-go会自动同步相关的dns信息到Cloudflare。

比如我在ddns-go的前端中设置域名为`ddnsv6.bakameow.xyz`，那么Cloudflare网页端就会有相关的DNS信息

现在就可以通过如下的CLI ssh命令直接通过域名访问家中的电脑了
```bash
ssh -6 <USER NAME>@<DOMAIN NAME>
```

可以更改`/etc/ssh/sshd_config`中的`Port`字段，更改ssh server绑定的端口。同时也建议设置ssh key仅允许密钥登录
```
Port 12345
...
PubkeyAuthentication yes
...
PasswordAuthentication no
```

## 远程开机
考虑到家庭服务器可能有远程开机的需求，故还是配置一下Linux平台的Wake-on-LAN。Wake-on-LAN的原理是关机时网卡仍在低功耗运行，在配置完主板BIOS的相关设置后，允许网卡接收到在魔术包（_Magic Packet_）后启动电脑。所以配置之前，需要保证家中有可控的其他设备，能发送魔术包来唤醒服务器。我有一台华子路由器支持远程操控发送魔术包。

```bash
sudo ethtool <DEVICE NAME> | grep -i "Wake-on"; # 查看网卡是否支持Wake-On-LAN
sudo ethtool -s "<DEVICE NAME>" wol g; # 设置网卡支持Wake-On-LAN
sudo vim /lib/systemd/system/wakeonlan.service;
sudo systemctl enable wakeonlan.service;
```

`wakeonlan.service`文件如此设定，当Linux启动时就会自动执行该该任务
```bash
[Unit]
Description=Enable Wake On Lan

[Service]
Type=oneshot
ExecStart=/usr/sbin/ethtool -s <DEVICENAME> wol g

[Install]
WantedBy=basic.target
```

## 定时关机
使用Linux系统自带的crontab事件管理器，当到达指定时间后执行相关命令

```bash
crontab -e; #第一次使用时还需要选择对应的文本编辑器
```

在文件的最后一行添加以下内容，使服务器会在每晚的23点准时执行`shutdown -h now`命令

```bash
0 23 * * * /sbin/shutdown -h now
```

通过下述命令查看定时任务是否设置成功

```bash
crontab -l
```
