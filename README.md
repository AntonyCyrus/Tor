# 个人自用Debian 12系统搭建Tor网络方案
## 更新软件源

```bash
sudo apt update
sudo apt upgrade
```

## 安装 Tor
使用 apt 包管理器安装 Tor

```bash
sudo apt install tor
```

## 启用 Tor 服务

```bash
sudo systemctl enable tor
```

```bash
sudo systemctl start tor
```

## 检查 Tor 是否正在运行

```bash
sudo systemctl status tor
```

## 配置 torrc 文件
Tor 的配置文件是 torrc，默认路径是 /etc/tor/torrc。如果您希望更改 Tor 的配置（比如修改端口、启用桥接等），请按照以下步骤

###  编辑 torrc 文件

```bash
sudo nano /etc/tor/torrc
```

###  个人推荐配置

```bash
# SOCKS5 代理监听本地 IPv4 
SocksPort 127.0.0.1:9050

# 启用桥接节点
ORPort 9001
BridgeRelay 1

# 禁止网桥公开，避免其他人自动获取该网桥
PublishServerDescriptor 0

# 禁止出口流量
ExitPolicy reject6 *:*, reject *:*

# 排除特定国家节点
ExcludeNodes {CN},{HK},{MO},{??}
ExitNodes {IS},{CH},{NO},{NL},{SE},{DK},{FI},{DE},{CA},{ES},{EE},{LV},{LT},{RO},{AR},{UY},{NZ}
StrictNodes 1  # 强制使用指定出口节点

Log notice file /var/log/tor/notices.log
```

保存修改并关闭编辑器。对于 nano，按 CTRL + O 保存，然后按 CTRL + X 退出

###  重启 Tor 服务

```bash
sudo systemctl restart tor
```

## 检查 Tor 是否正常运行
### 通过日志检查 Tor 状态
您可以查看 Tor 的日志，确认其是否启动并正常运行：

```bash
sudo journalctl -u tor -f
```

这将显示 Tor 的实时日志，您可以看到是否有任何错误或警告。

### 使用 torify 工具测试
Tor 安装完成后，您可以使用 torify 来测试是否成功通过 Tor 进行网络连接

```bash
torify curl https://check.torproject.org
```

### 检查 Tor 网络连接

```bash
nc -zv 127.0.0.1 9050
```

如果端口开启并监听，您应该看到类似以下输出：

```bash
Connection to 127.0.0.1 9050 port [tcp/socks] succeeded!
```

### 通过 Tor 测试互联网匿名性

```bash
torify curl ifconfig.me
```

您的 IP 应该会显示为 Tor 网络的出口节点的 IP，而不是您本地的 IP。

## 使用工具监测Tor流量

### nyx
如果您想要一个专门的监控工具来查看 Tor 网络的流量和状态，nyx（前身为 arm）是一个非常受欢迎的命令行工具，它为 Tor 提供了一个实时的监控界面。

```bash
sudo apt update
```

```bash
sudo apt install nyx
```

```bash
nyx
```

### torsocks
torsocks 是一个用于通过 Tor 网络透明代理应用程序流量的工具。它可以使普通的命令行工具（例如 curl 或 wget）通过 Tor 网络路由流量，而不需要手动配置这些工具。虽然它本身不专门用于监控 Tor 流量，但它可以让您确保通过 Tor 网络访问互联网。

```bash
sudo apt update
```

```bash
sudo apt install torsocks
```

```bash
torsocks curl -s https://check.torproject.org | grep -i "Congratulations"
```

这会通过 Tor 网络访问 check.torproject.org，并返回您是否在使用 Tor 网络的状态。

### torctl
torctl 是一个用于管理 Tor 进程的命令行工具，能够提供 Tor 网络的状态信息、启动和停止 Tor 服务、查看统计信息等。它可以帮助您监控 Tor 流量和连接的状态。

```bash
sudo apt update
```

```bash
sudo apt install torctl
```

```bash
torctl status
```

这个命令会返回关于 Tor 进程、节点连接、流量使用等信息。
