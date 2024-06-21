## **参考教程**

[如何优雅地访问远程主机？SSH与frp内网穿透配置教程_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV13L411w7XU/?spm_id_from=333.337.search-card.all.click&vd_source=10527fd74695c7dd4ae2589a62aa5f89)

[借公网ip-使用frp配置实验室服务器 - 染念Blog (dyedd.cn)](https://dyedd.cn/894.html)

↓ 以下内容搬运于 ↑

## frp简介

frp 是一个专注于内网穿透的高性能的反向代理应用，支持 TCP、UDP、HTTP、HTTPS 等多种协议。可以将内网服务以安全、便捷的方式通过具有公网 IP 节点的中转暴露到公网。

**使用场景**：例如，利用一个腾讯云租赁的轻量服务器作为公网IP节点，将学校内网服务器中的API服务暴露至公网

## 操作步骤

frp 主要由 客户端(frpc) 和 服务端(frps) 组成，服务端通常部署在具有公网 IP 的机器上，客户端通常部署在需要穿透的内网服务所在的机器上。

**注意：frpc和frps的端口号（即remote_port和server_port/bind_port）都要在具有公网IP的机器上开通，暴露至公网。因request需remote_port开放，客户端服务端连接需bind_port开放。**

1. 以腾讯云轻量服务器为例：
   1. 来到防火墙建立两个TCP端口

![image-20221119195832826.png](https://dyedd.cn/usr/uploads/2022/11/2105166807.png)

2. 目前可以在 Github 的 [Release](https://github.com/fatedier/frp/releases) 页面中下载到最新版本的客户端和服务端二进制文件，所有文件被打包在一个压缩包中。
   1. 一般我们的机器都是**AMD64**
   2. 下载的文件包含很多文件，建议分成2份
   3. 服务端：`frps.ini frps frps_full.ini`
   4. 客户端：`frpc.ini frpc frpc_full.ini`
   5. 在两个端都使用`sudo chmod 777 frpc`或者`sudo chmod 777 frps`更新权限，以防找不到命令

3. 在服务端，即轻量服务器，具有公网ip中，编辑frps.ini文件：
   1. 可以使用`./frps -c frps.ini`看看是不是端口呀，能不能启动

   ```bash
   [common]
   bind_port = 9960
   ```

4. 在客户端端，编辑frpc.ini文件：
   
   1. 可以使用`./frpc -c frpc.ini`看看两边能不能连接

```bash
[common]
server_addr = x.x.x.x#公网ip地址
server_port = 9960
[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 2022
```

如果有两台要连接frps的端口，此时可以在新的一台重复上述安装流程，此时[ssh]要改成[ssh1]，如果更多你可以使用[ssh2]、[ssh3]。。。

```ini
[common]
server_addr = x.x.x.x#公网ip地址
server_port = 9960
[ssh1]  # 若有多个客户端，名称不要重复。
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 6001  # 远程连接端口不要重复
```

5. 如果遇到类似这种问题：![image-20221119200649297.png](https://dyedd.cn/usr/uploads/2022/11/3773766645.png)
   1. 还有其它问题都看看下面的解决方式
   2. 就是两边的端口号写错了，跟你开启的防火墙不一样，不要相信网上增加什么配置，安装golang
   3. 如果还使用宝塔了，宝塔的安全也增加对应的端口

6. 如果出现`error: dial tcp 127.0.0.1:22: connect: connection refused`和`kex_exchange_identification: Connection closed by remote host`
   请先安装ssh

```bash
sudo apt update
sudo apt install openssh-server -y
# 如果你的防火墙开启了，使用下面语句
sudo ufw allow ssh
```

7. 因为两边都是开terminal的方式，不太好，而且得一直开着，这里推荐使用 `systemd` 控制 frps 及配置开机自启
   1. 使用文本编辑器，如 `vim` 创建并编辑 `frps.service` 文件。
      1. `vim /etc/systemd/system/frps.service`
      2. 写入内容

```bash
[Unit]
# 服务名称，可自定义
Description = frp server
After = network.target syslog.target
Wants = network.target

[Service]
Type = simple
# 启动frps的命令，需修改为您的frps的安装路径
ExecStart = /path/to/frps -c /path/to/frps.ini

[Install]
WantedBy = multi-user.target
```

**怎么更好看安装路径呢？在你的解压目录，使用`pwd`，直接将输出的结果cv一下即可。**

使用 `systemd` 命令，管理 frps。

```bash
# 启动frp
systemctl start frps
# 停止frp
systemctl stop frps
# 重启frp
systemctl restart frps
# 查看frp状态
systemctl status frps
```

配置 frps 开机自启。

```bash
systemctl enable frps
```

frpc类同！省略,但请在[service]下面添加

```ini
Restart=on-failure
RestartSec=5s
```

因为frpc是自己的机器，有时候自己的机器没有连网，就会出现连接错误！

这样服务端和客户端都配置好，就能开始偷偷的卷了...

## 连接方式

我们根据frpc的remote_port，进行如下命令连接：
`ssh -oPort=<remote_port> <用户名>@服务器ip`

## 进阶-本地打开服务器浏览器端口

有时候，我们在远程服务器上通过一些服务打开了浏览器端口，这时其实是可以映射到本地端口的。
我们可以使用SSH隧道来实现。假设已经成功地通过FRP进行了内网穿透，就可以使用以下命令在本地命令行中输入：

```
ssh -L 8887:localhost:8888 -p <remote_port> <用户名>@服务器ip
```

第一个8887是在本地浏览器打开的，第二个8888是在远程服务器上启动的端口。
这样，我们可以在本地浏览器中访问[http://localhost:8887](http://localhost:8887/)，实际上是访问服务器上的8888端口服务。

## 进阶-本地访问内网服务

在之前的基础上，简单修改配置文件即可

### **服务端**

配置文件不用变

```bash
[common]
bind_port = 9960 # 与客户端绑定的端口
```

### **客户端**

修改下配置文件

**本质**：让服务端（公网IP节点）的`remote_port`端口的网络请求转发至客户端（内网服务器）的服务端口`local_port`即可

```bash
[common]
server_addr = x.x.x.x#公网ip地址
server_port = 9960
[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 9001 # 修改此处为 内网服务器的API服务端口（例9001)
remote_port = 2022 # 服务端暴露在公网上的端口，服务端remote_port的请求均转发至客户端的local_port
```

### **启动**

先启动服务端frp（公网IP节点），后启动客户端frp（内网服务器）

**注意：frpc和frps的端口号（即remote_port和server_port/bind_port）都要在具有公网IP的机器上开通，暴露至公网。因request需remote_port开放，客户端服务端连接需bind_port开放。**

### **request实例**

`GET/POST  http://{公网IP}:{remote_port}/内网服务API名称`

