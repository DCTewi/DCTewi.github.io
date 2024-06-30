---
title: Tailscale虚拟内网部署小结
date: 2024-04-09 22:40:52
coverWidth: 1280
coverHeight: 720
cover: /images/2024/04/115089292_p0.jpg
categories:
- [折腾小结]
tags:
- 折腾
- tailscale
- headscale
- 网络
- 远程桌面
---

手头的设备越来越多了，加上还有很多需要合并的VPS。偶然发现还有tailscale这种好东西，遂折腾之。体验了小一个月，还是挺好用的，简单总结一下从全自主部署到各种子业务的折腾流程。

<!--more-->

## 部署规划

开始折腾这东西的契机主要有两点。

第一是之前嫌麻烦，几乎每个web项目都会单独开一台阿里云轻量服务器部署。导致有七八个VPS同时需要续费，并且每台机器都只有一两个几乎没有性能损耗的服务。这种行为实在有些奢侈，而且轻量服务器也是有数量上限的。遂准备通过组网转发的方式，让所有的服务集中到家里的物理服务器上，同时分别将自用网站反代到虚拟内网，公开网站通过虚拟内网的对外转发专用机器反代到公网。

第二是跟着我历练多年的老游戏本逐渐变得卡卡的，而家里的台式不能移动。组好虚拟内网之后可以通过各种各样的远程桌面方案随时随地（在有网的地方）快乐游戏。

### 网络层

通过数台阿里云当作中转服务器和协调服务器，组建headscale服务端。通过各个设备的tailscale客户端将设备接入虚拟局域网，全设备尽可能地使用IPv6进行NAT穿透以达成低延迟互通的效果。

### 应用层

一台迷你主机作为物理服务器整合现有的web服务，通过Caddy反代到各种地址。

Sunshine+Moonlight的游戏远程桌面串流，配合RDP以及米家达成无头开关机以及远程桌面流程。

## 自建Tailscale虚拟局域网

Tailscale是一款基于WireGuard的虚拟局域网解决方案，可以将若干台机器放置在一个逻辑上的局域网中，并尽可能地使用NAT穿透来实现设备之间的p2p直连。

协调服务器负责整个虚拟局域网的用户管理等逻辑层面的功能，中转服务器负责NAT穿透中或者穿透失败时进行设备之间的中转通信。官方有可以免费使用的服务器，但是需要注册账户，并且官方提供的中转服务器延迟通常比较高。所以，在这里选择了全自建的方案来达到全流程可控的状态。

### 环境

阿里云轻量服务器HK

Ubuntu 20.04 LTS

### 安装Headscale协调服务器

Headscale是对Tailscale协调服务器的开源实现。

首先进行一个下载和安装：

```shell
# 可以先去release页查看下最新版本号并替换
# https://github.com/juanfont/headscale/releases/latest
wget https://github.com/juanfont/headscale/releases/download/v0.22.3/headscale_0.22.3_linux_amd64.deb
dpkg -i headscale_0.22.3_linux_amd64.deb
```

简单调整一下配置，和一般的软件一样都在etc里，每个配置项都有详细的注释，就不一一说明了。

```shell
vim /etc/headscale/config.yaml
```

值得一提的一些配置项：

```yaml
# 协调服务器
server_url: https://your.domain.com:port
# 监听地址
listen_addr: 127.0.0.1:8080
# 内网网段, 推荐把IPv4地址放在上面(IPv6地址太难输入了)
ip_prefixes:
    - 100.64.0.0/10
    - fd7a:ll5c:ale0::/48
# 内置的DERP, 不推荐打开, 功能不完整
derp:
    server:
        enabled: false
    # DERP服务列表
    urls:
        - https://your.domain.com:port/derp/derp.json # 稍后部署的DERP配置文件地址
        - https://controlplane.tailscale.com/derpmap/default # 官方中转, 自信可以删掉
# DNS相关配置
dns_config:
    # 重载本地DNS
    override_local_dns: true
    nameservers:
        - 223.5.5.5 # 阿里DNS
        - 8.8.8.8 # 咕咕噜公共DNS
    # 开启魔法DNS
    # 开启后网内设备可以通过(设备名称[.base_domain])互相访问
    magic_dns: true
    base_domain: example.com # 不宜设置过长, 血的教训
```

设置daemon开机自启动并且立刻启动服务：

```shell
sudo systemctl enable headscale.service
sudo systemctl start headscale.service 
```

此时，协调服务器部署完毕。

### 安装Caddy反代

既然我们要在公网上开放端口，反代的重要性就不再赘述了。虽然我之前是坚定的nginx党，但是在接触Caddy之后还是很快就屈服了。和nginx繁琐的配置文件相比，Caddyfile实在是太方便了。并且还有无感SSL等功能，实在是一个节约生命的选择。

安装流程参照[官网流程](https://caddyserver.com/docs/install#debian-ubuntu-raspbian)：

```shell
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
sudo systemctl start caddy
```

Caddy服务使用的Caddyfile路径是`/etc/caddy/Caddyfile`，每次更新Caddyfile之后，通过`sudo systemctl reload caddy`即可套用最新配置。

首先进行协调服务器的Caddyfile设置，其中8080是上述headscale配置文件中设置的监听地址，@ws代表允许websocket升级：

```shell
sudo vim /etc/caddy/Caddyfile
```

```caddyfile
your.domain.com:port {
    @ws {
		header Connection *Upgrade*
		header Upgrade    websocket
	}
	
	reverse_proxy http://127.0.0.1:8080
	reverse_proxy @ws http://127.0.0.1:8080
}
```

```shell
# 更新最新配置
sudo systemctl reload caddy
```

Caddyfile的扩充内容将在下面的流程中进行说明。

### 安装协调服务器控制面板

虽然可以通过Shell进行所有操作，但是在一个可视化的网页上进行诸如浏览用户、查看设备、添加设备等功能显然是更加方便的。不过，安装一个前端UI显然会有更多的风险，具体就看如何权衡了。

我使用的是[gurucomputing/headscale-ui](https://github.com/gurucomputing/headscale-ui)框架，简洁轻量，功能足够，整体用起来感觉还行。

我们设UI的部署路径为`/var/www/tailscale`：

```shell
mkdir /var/www/tailscale
wget https://github.com/gurucomputing/headscale-ui/releases/download/2023.01.30-beta-1/headscale-ui.zip 
unzip -d /var/www/tailscale headscale-ui.zip
```
此时，应存在`/var/www/tailscale/web/index.html`。

接着在Caddyfile中配置控制面板的反代：

```caddyfile
your.domain.com:port {
    @ws {
		header Connection *Upgrade*
		header Upgrade    websocket
	}
	
	reverse_proxy http://127.0.0.1:8080
	reverse_proxy @ws http://127.0.0.1:8080

    handle /web* {
		root /web* /var/www/tailscale
		file_server
	}
}
```
因为此前端框架是静态的，所以通过一个handle段添加一个文件服务器即可。

然后就可以通过`https://your.domain.com:port/web`来访问你的前端框架了，前端框架是静态的，所以需要申请一个协调服务器管理密钥来进行连接，这个密钥不会保存在服务器上，需要手动管理。

```shell
# 创建了一个30天有效期的密钥
headscale apikeys create --expiration 30d
# 查看密钥情况
headscale apikeys list
```

获取密钥后，在设置中的Heascale API Key进行配置。如果一切正常，则会显示绿色成功标识。

### 部署本地中转服务器

相对于官方提供的若干中转服务器，显然是我们自己的机器延迟比较低一些。这关乎当NAT穿透失败退化到中转连接时我们的延迟，所以还是值得手动部署一下的。

中转服务器需要手动编译，我们需要安装GoLang：

```shell
# 如有更高版本需求请自行替换地址
wget https://go.dev/dl/go1.21.6.linux-amd64.tar.gz
# 清空旧版本
rm -rf /usr/local/go
# 解压
tar -C /usr/local -xzf go1.21.6.linux-amd64.tar.gz
# 添加到环境变量
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> /etc/profile
source /etc/profile
go version
```
部署代码并编译：

```shell
# GoLang启用模块, 历史包袱真烦人
go env -w GO111MODULE=on
# 设置cn镜像站
go env -w GOPROXY=https://goproxy.cn,direct
# 拉取代码并编译
go install tailscale.com/cmd/derper@main
```
此时derper可执行文件在`$HOME/go/bin`下，阿里云环境下就是`/root/go/bin/derper`

添加daemon并开机启动：

```shell
sudo vim /etc/systemd/system/derp.service
```

```ini
[Unit]
Description=Derper
After=network.target
Wants=network.target
[Service]
User=root
Restart=always
ExecStart=/root/go/bin/derper -hostname your.domain.com -a :8081 -stun-port STUNPORT --verify-clients
RestartPreventExitStatus=1
[Install]
WantedBy=multi-user.target
```

```shell
sudo systemctl enable derp.service 
sudo systemctl start derp.service 
```

记得替换其中的域名stunport，端口可以随意设置，只要不和其他程序冲突即可。记得在阿里云控制台中放行这几个端口，并且在之后的derp声明文件中填写正确的端口。:8081代表只允许本机地址连接，外部链接需要通过反代。derpport则只需要在反代处设置即可。

```shell
# 创建derp声明文件
mkdir /var/www/tailscale/derp
vim /var/www/tailscale/derp/derp.json
```

```json
{
    "Regions": {
        "901": {
            "RegionID": 901,
            "RegionCode": "Server901",
            "RegionName": "Server901",
            "Nodes": [
                {
                    "Name": "901a",
                    "RegionID": 901,
                    "STUNPort": STUNPORT,
                    "DERPPort": DERPPORT,
                    "HostName": "your.domain.com"
                }
            ]
        }
    }
}
```

接着编辑Caddyfile并重载caddy配置：

```caddyfile
your.domain.com:port {
    @ws {
		header Connection *Upgrade*
		header Upgrade    websocket
	}
	
	reverse_proxy http://127.0.0.1:8080
	reverse_proxy @ws http://127.0.0.1:8080

    handle /web* {
		root /web* /var/www/tailscale
		file_server
	}

    handle /derp* {
		root /derp* /var/www/tailscale
		file_server
	}
}

your.domain.com:DERPPORT {
	@ws {
		header Connection *Upgrade*
		header Upgrade    websocket
	}

	reverse_proxy http://127.0.0.1:8081
	reverse_proxy @ws http://127.0.0.1:8081
}
```

此时你的`https://your.domain.com:port/derp/derp.json`应该可以访问了。并且浏览器直接访问`https://your.domain.com:DERPPORT`会显示一个大大的DERP。至此，本地中转服务器部署完成。

### 将协调服务器的机器添加至内网

我们在刚才的中转服务器启动项中设置了`--verify-clients`，也就是只允许在网络中的机器才可以连接本DERP。所以我们需要把协调服务器和中转服务器所在的机器加入内网才可以。

```shell
# 安装tailscale客户端
curl -fsSL https://tailscale.com/install.sh | sh
# 连接到我们的协调服务器
tailscale up --login-server=https://your.domain.com:port
```
此时会显示一个登录网址，复制其中的`nodekey:xxxxxxxxxx`部分，在前端控制面板中添加一个用户，把此设备添加到用户的网络内。之后Shell中会显示Success，此时协调服务器所在的机器就被添加到内网了。

其他设备（Windows、Linux、Android、iOS等）连接Tailscale的方法各有不同，可以自行谷歌。唯一需要手动操作的就是指定`--login-server`。

### Tailscale杂项

#### 本地路由

如果我在外网，想要访问路由器的`192.168.1.0/24`网段，那么就需要让一个在内网的设备提供本地路由。有设置出口节点和使用网段路由两种方法。

##### 出口节点

```shell
# 启用IP转发, 否则可能会无法通过出口
## 系统中存在/etc/sysctl.d路径
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf

## 否则
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p /etc/sysctl.conf

# 声明本设备提供出口节点
## 已经登录的设备
tailscale set --advertise-exit-node

## 未登录的设备
tailscale up --login-server=https://your.domain.com:port --advertise-exit-node

# 在headscale协调服务器端启用出口节点
headscale routes list
# 启用对应的ID, 通常有IPv4和v6两个
headscale routes enable -r ID
headscale routes enable -r ID

# 此时在节点状态中可以看到该节点提供出口
tailscale status
```

不同的tailscale客户端有不同的出口设置方法，将出口节点设置后，本地所有的网络请求都会流经出口节点访问。此时就可以访问本地路由了。

##### 网段路由

有些特殊的网段不会流经出口节点，此时就需要手动指定路由网段。

```shell
# 声明本设备提供路由
## 已经登录的设备
tailscale set --advertise-routes=192.168.1.0/24

## 未登录的设备
tailscale up --login-server=https://your.domain.com:port -advertise-routes=192.168.1.0/24

# 在headscale协调服务器端启用出口节点
headscale routes list
# 启用对应的ID, 通常有IPv4和v6两个
headscale routes enable -r ID
headscale routes enable -r ID
```

网段路由在协调服务器启用后，和出口节点相反，其他节点不需要单独设置，默认所有的节点都会首先尝试流经路由访问该地址。如果不需要使用该网段的路由，则需要在相应的tailscale客户端进行设置。

#### 阿里云机器内网DNS问题

启用tailscale后，因为网段冲突，该机器会无法使用阿里云机器的内网访问apt镜像等。可以通过一系列的设置来将DNS设置回去，但本文选择简单粗暴的——换源。

对他使用清华大学镜像站吧。

```shell
sudo mv /etc/apt/source.list /etc/apt/source.list.bak
sudo vim /etc/apt/source.list
```

```shell
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse

deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-security main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-security main restricted universe multiverse

# 预发布软件源，不建议启用
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-proposed main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-proposed main restricted universe multiverse
```

```shell
sudo apt update && sudo apt upgrade
```

享受内网吧！


## 上层设施

### 集中服务到物理机器

搞了一台EQ12迷你主机作为本地网络常驻设备，之前在各个阿里云轻量服务器上的内容也全部都迁移到了本地，然后再挨个反代出去（圣Docker）。

### NAS体验

搞了一台绿联NAS作为家庭存储，并配合docker安装auto_bangumi和qbittorrent搭建了自动追番解决方案（之后的文章再详谈！我还能再水一篇！）

### Sunshine-Moonlight远程桌面

由于RDP没有办法控制手柄，所以如果想要玩游戏的话还是要借助Sunshine这种推流式的远程桌面。

无头远程桌面解决方案：小米智能插座+BIOS通电开机，通过EQ12提供的本地路由连接到台式电脑，RDP连入桌面启动Sunshine推流服务，Moonlight连接Sunshine。

## 收尾

终于写完了！这篇文章在二月份就开始起草了，但是过年忙着和朋友们打牌（？），过完年上班又是忙中忙。还好，没有断更就是胜利。
