
## 0x00 前言

&emsp;&emsp;年初买了一台台式电脑，本想着买来玩玩游戏，后来也确实玩了不少单机（老头环是真难打），但是因为工作的原因，很多时候都是回家就想睡觉，周末的时候也没有想玩游戏的动力了，除非几个好友跟我一起打LOL（感觉得了电子阳痿）。这个台式前前后后也加了不少配置，现在有48G内存2T的固态，想想也不能白买了，得充分用上这个配置，而我又喜欢折腾些花里胡哨的东西，所以想起来整一套家庭网络，把这个台式机没用的时候当一台服务器来用，当然后续还会接一些其他设备进来，最后完善我自己的家庭网络。


## 0x01 准备

&emsp;&emsp;家里面办了中国移动的1000M的宽带，而中国移动现在推行IPv6，所以所有的宽带用户都是免费办理IPv6。

<img src="images/Pasted image 20230304140657.png" width="auto"  height="auto"/>

&emsp;&emsp;那么公网IP地址有了，中国移动给的IPv6地址是动态的，具体几天变一次我没有去统计，但是觉得是会变的，所以后续还需要准备一个DDNS解析。我在做准备之前也有在网上搜索了关于Openwrt搭建用IPv6搭建VPN的相关文章，后续会在参考文章一一添加。

&emsp;&emsp;考虑到用哪种VPN的时候我这边有两个选择，一是openVPN，二是今天的主角WireGuard。openVPN和WireGuard之间我选择WG（后续称WireGuard为WG）的原因是因为配置简单（在Google上我没有找到使用IPv6搭建openVPN的相关文章，当然其实跟IPv4搭建基本没啥区别，但是能有现成的文章我接下来做事也更有底气吧，而我找到了使用IPv6搭建WG的文章和视频），而且因为WG使用的加密算法，以及使用UDP传输所以传输的速度相比之下是更快的，在这里其实我有一个担忧就是在国内环境里，IPv4的UDP传输的Qos是比较差的，三大运营商都会“杀”UDP流量，导致用UDP做VPN隧道的效果不是很好，所以网上有一种方法就是给UDP再套一层TCP，其实到这已经知道了，这种传输效率会大大降低。我后续也问了一些在中国移动工作的朋友，以及一些用IPv6搭建过WG的朋友，都说IPv6的Qos对UDP还行，速率也挺快的。

&emsp;&emsp;把公网IP和VPN准备好，我这边准备把OpenWrt安装到路由器上，因为第一次刷路由器，为了不承担更多的经济成本，我直接买了一个自带OpenWrt的路由器`GL-AX1800`，并且这个路由器也可以自己刷固件，而且还保修。

<img src="images/Pasted image 20230304144624.png" width="auto"  height="auto"/>

## 0x02 开始安装

### 安装WireGuard插件
- wireguard-tools
- kmod-wireguard
- luci-app-wireguard
- luci-i18n-wireguard-zh-cn
- luci-proto-wireguard (开启图形二维码的插件)

>如果没有找到上述插件，请更新软件包列表

### 创建公钥和私钥以及预共享密钥

- **创建预共享密钥**

```shell
mkdir wg
# 创建目录存放公钥私钥
cd wg
# 进入文件夹
umask 077
# 配置创建密钥的权限
wg genpsk > sharekey
# 创建预共享密钥
cat sharekey
# 获取密钥复制保存
```

- **创建服务器密钥**

```shell
wg genkey | tee server_privatekey | wg pubkey > server_publickey
# 创建服务端公钥和私钥
cat server_privatekey
# 获取服务端私钥复制保存
cat server_publickey
# 获取服务端公钥复制保存
```

- **创建客户端密钥**

```shell
wg genkey | tee windows_privatekey | wg pubkey > windows_publickey
# 创建 Windows 客户端公钥和私钥
cat windows_privatekey
# 获取 Windows 客户端私钥复制保存
cat windows_publickey
# 获取 Windows 客户端公钥复制保存
```

>这里的客户端的话，如果还要使用手机来访问就需要为手机客户端创建密钥

### 创建WireGuard防火墙

选择网络 -> 防火墙 -> 常规设置 -> 新增

<img src="images/Pasted image 20230304152411.png" width="auto"  height="auto"/>

<img src="images/Pasted image 20230304152444.png" width="auto"  height="auto"/>

选择网络 -> 防火墙 -> 通信规则 -> 新增

<img src="images/Pasted image 20230304152610.png" width="auto"  height="auto"/>

<img src="images/Pasted image 20230304152701.png" width="auto"  height="auto"/>

<img src="images/Pasted image 20230304152720.png" width="auto"  height="auto"/>

### 创建WireGuard接口

选择网络 -> 接口 -> 添加新接口

<img src="images/Pasted image 20230304151643.png" width="auto"  height="auto"/>

<img src="images/Pasted image 20230304152028.png" width="auto"  height="auto"/>

设置MTU为1280

<img src="images/Pasted image 20230304152818.png" width="auto"  height="auto"/>

防火墙设置为上述的`wgserver`

<img src="images/Pasted image 20230304152849.png" width="auto"  height="auto"/>

### 创建DDNS

**安装DDNS插件**

- luci-i18n-ddns-zh-cn

当然还有就是要考虑到用哪家的DDNS服务就装哪家的插件，比如cloudflare或者cnkuai。

我这里使用的是[dynv6](https://dynv6.com/)家的DDNS服务。

<img src="images/Pasted image 20230304153352.png" width="auto"  height="auto"/>

这里用邮箱去创建一个账户，然后通过邮件进行激活（因为这个网站没有激活也能进行操作，我就没激活，后面发现只有激活了它才给我解析DNS）。

这里选择`dynv6.net`这个域名，听说是国内解析快点，然后起一个随机一点的名字别和别人重名了。

<img src="images/Pasted image 20230304154137.png" width="auto"  height="auto"/>

然后在`ddclient`获取到一些关键信息

```
protocol=dyndns2
server=dynv6.com
login=none
password='H6su2ZxmBd6b8KezG4hz_y2BSWm7eb'
pubpng123.dns.army
```

选择服务 -> 动态DNS -> 添加新服务

<img src="images/Pasted image 20230304154336.png" width="auto"  height="auto"/>

<img src="images/Pasted image 20230304154611.png" width="auto"  height="auto"/>

<img src="images/Pasted image 20230304154921.png" width="auto"  height="auto"/>

<img src="images/Pasted image 20230304154948.png" width="auto"  height="auto"/>



至此服务端的内容就算是设置完毕了。

### 设置客户端

选择之前创建的WG接口，选择对端

<img src="images/Pasted image 20230304155539.png" width="auto"  height="auto"/>

然后在[WireGuard官网下载windows客户端](https://download.wireguard.com/windows-client/)

配置文件如下

```
[Interface]
Address = 10.10.10.3/32
# 对应 Windows 客户段分配的 IP
PrivateKey = <windows_privatekey>
DNS = 192.168.1.3
# 本地的 DNS 服务器或者公有 DNS 服务器,例如: 114.114.114.114
[Peer]
PublicKey = <server_publickey>
AllowedIPs = 192.168.1.0/24, 192.168.100.0/24, 10.10.10.0/24
# iPhone iPad 设置为 0.0.0.0/0 全局则模式.
PresharedKey = <sharekey>
# 预共享密钥
Endpoint = <DDNS Domain>:<Port>
PersistentKeepalive = 25
```

连接上的效果如下

<img src="images/Pasted image 20230304160145.png" width="auto"  height="auto"/>

苹果手机的话需要下载WireGuard客户端（需要外区APP Store账号），配置和上面一样的。


## 0x03 结语

&emsp;&emsp;折腾了两个完善总算是把问题解决了，实际上遇到问题挺多的，尤其是第一次接触OpenWrt和WireGuard。还有就是最好不要折腾路由器，因为路由器大部分都是arm架构的，而且类型很多，中间遇到很多版本的坑点，所以说能用x86就用x86，最好还是有一台软路由，我那台路由器也就512的内存，虽然说我自己挂了一个U盘在上面，实际上因为arm架构的生态问题，想玩的一些东西目前也用不了。


## 0x04 参考文章


- [OpenWRT 配置 WireGuard 服务端及客户端配置教程](https://www.ioiox.com/archives/143.html)
- [WireGuard on OpenWrt+IPv6 组网](https://www.whosneo.com/wireguard-openwrt-ipv6/)
- [免费好用的IPv6之DDNS服务-Openwrt上dynv6的使用介绍](https://blog.csdn.net/weixin_43593122/article/details/95660085)







