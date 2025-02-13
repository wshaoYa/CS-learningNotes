## **参考教程**

[SSH原理与运用（二）：远程操作与端口转发 - 阮一峰的网络日志 (ruanyifeng.com)](https://www.ruanyifeng.com/blog/2011/12/ssh_port_forwarding.html)

↓ 以下内容搬运于 ↑

## **绑定本地端口**

既然SSH可以传送数据，那么我们可以让那些不加密的网络连接，全部改走SSH连接，从而提高安全性。

假定我们要让8080端口的数据，都通过SSH传向远程主机，命令就这样写：

``` bash
ssh -D 8080 user@host
```

SSH会建立一个socket，去监听本地的8080端口。一旦有数据传向那个端口，就自动把它转移到SSH连接上面，发往远程主机。可以想象，如果8080端口原来是一个不加密端口，现在将变成一个加密端口。

## **本地端口转发**

有时，绑定本地端口还不够，还必须指定数据传送的目标主机，从而形成点对点的"端口转发"。为了区别后文的"远程端口转发"，我们把这种情况称为"本地端口转发"（Local forwarding）。

假定host1是本地主机，host2是远程主机。由于种种原因，这两台主机之间无法连通。但是，另外还有一台host3，可以同时连通前面两台主机。因此，很自然的想法就是，通过host3，将host1连上host2。

我们在host1执行下面的命令：

```bash
ssh -L 2121:host2:21 host3
```

命令中的L参数一共接受三个值，分别是"本地端口:目标主机:目标主机端口"，它们之间用冒号分隔。

**这条命令的意思，就是指定SSH绑定本地端口2121，然后指定host3将所有的数据，转发到目标主机host2的21端口（假定host2运行FTP，默认端口为21）。**

这样一来，我们只要连接host1的2121端口，就等于连上了host2的21端口。

```bash
ftp localhost:2121
```

"本地端口转发"使得host1和host3之间仿佛形成一个数据传输的秘密隧道，因此又被称为"SSH隧道"。

下面是一个比较有趣的例子。

```bash
ssh -L 5900:localhost:5900 host3
```

**它表示将本机的5900端口绑定host3的5900端口（这里的localhost指的是host3，因为目标主机是相对host3而言的）。**

另一个例子是通过host3的端口转发，ssh登录host2。

```bash
ssh -L 9001:host2:22 host3
```

这时，只要ssh登录本机的9001端口，就相当于登录host2了。

```bash
ssh -p 9001 localhost
```

上面的-p参数表示指定登录端口。

## **远程端口转发**

既然"本地端口转发"是指绑定本地端口的转发，那么"远程端口转发"（remote forwarding）当然是指绑定远程端口的转发。

还是接着看上面那个例子，host1与host2之间无法连通，必须借助host3转发。但是，特殊情况出现了，host3是一台内网机器，它可以连接外网的host1，但是反过来就不行，外网的host1连不上内网的host3。这时，"本地端口转发"就不能用了，怎么办？

解决办法是，既然host3可以连host1，那么就从host3上建立与host1的SSH连接，然后在host1上使用这条连接就可以了。

我们在host3执行下面的命令：

```bash
ssh -R 2121:host2:21 host1
```

R参数也是接受三个值，分别是"远程主机端口:目标主机:目标主机端口"。

**这条命令的意思，就是让host1监听它自己的2121端口，然后将所有数据经由host3，转发到host2的21端口。由于对于host3来说，host1是远程主机，所以这种情况就被称为"远程端口绑定"。**

绑定之后，我们在host1就可以连接host2了：

```bash
ftp localhost:2121
```

这里必须指出，"远程端口转发"的前提条件是，host1和host3两台主机都有sshD和ssh客户端。

## **SSH的其他参数**

SSH还有一些别的参数，也值得介绍。

N参数，表示只连接远程主机，不打开远程shell；T参数，表示不为这个连接分配TTY。这个两个参数可以放在一起用，代表这个SSH连接只用来传数据，不执行远程操作。

```bash
ssh -NT -D 8080 host
```

f参数，表示SSH连接成功后，转入后台运行。这样一来，你就可以在不中断SSH连接的情况下，在本地shell中执行其他操作。

```bash
ssh -f -D 8080 host
```

要关闭这个后台连接，就只有用kill命令去杀掉进程。

## **实例1**

公网访问内网API服务

**前提**：需一个公网服务器做中介

1. ssh登录内网服务器，将公网服务器远程端口转发至本地localhost

```bash
ssh -NRf {公网服务器端口}:localhost:{内网服务器API端口} {username}@{公网IP}
```

2. 公网服务器nginx反向代理，将用户请求转发至{公网服务器端口}处即可

![image-20240425203858707](https://s2.loli.net/2024/04/25/I9qPEa5pZshk8DR.png)