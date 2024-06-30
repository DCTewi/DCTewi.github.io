---
title: 使用Caddy转发内网的Minecraft服务器小结
coverWidth: 1280
coverHeight: 720
date: 2024-06-30 15:24:17
cover: /images/2024/06/1719504530134.jpeg
categories:
- [折腾小结]
tags:
- 折腾
- tailscale
- 网络
- caddy
- minecraft
---

接上文，我搭建了一个tailscale虚拟内网，把自己的各种设备在一个虚拟网段中组织了起来。于是，通过一个拥有外网IP的节点，就可以把自己内网的Minecraft服务器转发到外网，算是一个内网穿透的新选择。

<!--more-->

相比nginx，我更加喜欢配置简单清晰的caddy。而且还有最新最潮的自动SSL功能，实在是降低心智负担的好选择。但是，caddy和nginx都属于HTTP层的转发代理。众所周知，Minecraft使用了自定义的TCP协议进行网络通信，所以无论是nginx还是caddy，都需要通过安装额外插件的方式来支撑TCP转发。

## xcaddy

xcaddy（[caddyserver/xcaddy: Build Caddy with plugins (github.com)](https://github.com/caddyserver/xcaddy)）是一个编译、调试和管理caddy插件的项目。也是基于Go，所以我们需要先安装一个Go环境。

按照[Go官方推荐的安装方法](https://go.dev/doc/install)，我们把go环境（也就是GOROOT）安装到`/usr/local/go`，默认的GOPATH则是在`~/go`。

```shell
wget https://go.dev/dl/go1.22.4.linux-amd64.tar.gz
rm -rf /usr/local/go
tar -C /usr/local -xcf go1.22.4.linux-amd64.tar.gz
echo "export PATH=$PATH:/usr/local/go/bin" | tee -a /etc/profile
go version
```

此时go已经安装完毕，打开go111module选项：

```shell
go env -w GO111MODULE=auto
```

接着编译安装xcaddy：

```shell
go install github.com/caddyserver/xcaddy/cmd/xcaddy@latest
```

此时xcaddy可执行文件被安装到了`~/go/bin/xcaddy`

## caddy-l4

caddy-l4（[mholt/caddy-l4: Layer 4 (TCP/UDP) app for Caddy (github.com)](https://github.com/mholt/caddy-l4)）是caddy的一个TCP层代理插件，可以让caddy支持TCP/UDP/SOCKS等第四层的网络代理。

caddy的插件都将通过源码的方式编译到二进制文件中，也就是说我们需要通过xcaddy编译一个具有caddy-l4插件的二进制文件并替换我们已有的caddy可执行文件：

```shell
~/go/bin/xcaddy build --with github.com/mholt/caddy-l4
```

等待编译结束，此时带有caddy-l4插件的caddy将会出现在当前路径中，我们使用这个caddy将原来的caddy替换。

```
mv caddy /usr/bin/caddy
systemctl restart caddy
```

此时caddy-l4安装完成，接下来我们来配置caddy。


## 使用JSON配置文件

鉴于caddy-l4模块目前并不支持Caddyfile，所以我们将使用JSON格式的配置文件来运行caddy。不过该模块的Caddyfile支持也是在不断地推进当中了，不远的将来也许我们可以继续使用Caddyfile（[Feature Request: Support Caddyfile · Issue #16 · mholt/caddy-l4 (github.com)](https://github.com/mholt/caddy-l4/issues/16)）。

caddy的JSON格式配置文件的说明可以从[官网](https://caddyserver.com/docs/json/)查看，我们可以通过以下指令将Caddyfile转换到JSON格式的配置文件：

```shell
caddy adapt --config /etc/caddy/Caddyfile --pretty >> /etc/caddy/caddy.json
```

可以看到官方默认的代理都在`apps.http`中，caddy-l4的配置将在`app.layer4`中，格式和http类似。具体的matcher和handler列表可以在github项目主页看到（但很可惜并没有相关说明，具体的使用方法只能通过看代码或者通过灵感检定获知）。

我的minecraft服务器在tailscale内网的100.64.0.17:25565端口，于是可以这样编写配置文件：

```json
{
    "apps": {
        "layer4": {
            "servers": {
                "minecraft": {
                    "listen": ["0.0.0.0:23343"],
                    "routes": [
                        {
                            "handle": [
                                {
                                    "handler": "proxy",
                                    "upstreams": [
                                        { "dial": ["100.64.0.17:25565"] }
                                    ]
                                }
                            ]
                        }
                    ]
                }
            }
        }
    }
}
```

此配置文件将监听23343端口的TCP请求，并将其转发到100.64.17的25565端口。

更改caddy服务使用的启动选项：

```ini
# /usr/lib/systemd/system/caddy.service
# 其他选项省略
[Service]
ExecStart=/usr/bin/caddy run --environ --config /etc/caddy/caddy.json
ExecReload=/usr/bin/caddy reload --config /etc/caddy/caddy.json --force
```

重载并重启caddy服务：

```shell
systemctl daemon-reload
systemctl restart caddy
```

在Minecraft中把指定IP的23343端口添加到服务器列表并刷新，可以看到当前已经联通（别忘了在VPS控制台放开指定端口）。查看caddy的日志可以发现确实收到了查询服务器状态的请求：

```
journalctl -u caddy
```

## 设置SRV转发

通过SRV转发，可以让访问Minecraft服务器时省略掉指定端口。

以腾讯云DNS为例，将`_minecraft._tcp.mc`记录转发到`0 5 23343 xxx.xxx.xxx.xxx.`。含义是将通过mc子域名访问的minecraft协议tcp请求转发到xxx.xxx.xxx.xxx:23343，优先级是0，权重是5。

等待DNS缓存服务器刷新。然后就可以使用mc.yourdomain.com访问Minecraft服务器了。

也可以通过以下windows指令检查SRV解析情况：

```powershell
nsloopup.exe -q=srv _minecraft._tcp.mc.yourdomain.com
```

到此就大功告成了！可喜可贺，可喜可贺。
