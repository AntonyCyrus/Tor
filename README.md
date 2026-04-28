Debian 13 从 0 安装 Tor（最小可用教程）

一、安装 Tor
更新
```bash
apt update && apt upgrade -y
apt install apt-transport-https curl gnupg2 -y
```
添加 Tor Project 官方存储库
添加 GPG 密钥
```bash
curl -f sS https://deb.torproject.org/torproject.org/A3C4F0F979CAA22CDBA8F512EE8CBC9E886DDD89.asc | gpg --dearmor | tee /usr/share/keyrings/tor-archive-keyring.gpg >/dev/null
```
添加存储库（针对 Debian 13）：
```bash
echo "deb [signed-by=/usr/share/keyrings/tor-archive-keyring.gpg] https://deb.torproject.org/torproject.org trixie main" | tee /etc/apt/sources.list.d/tor.list
```
安装 Tor 和 obfs4proxy
```bash
apt update
apt install tor deb.torproject.org-keyring obfs4proxy -y
```

二、确认服务状态
```bash
systemctl status tor --no-pager
```
期望看到：
Active: active (running)

三、配置 Tor

编辑配置文件：
```bash
cat <<EOF > /etc/tor/torrc
# ==========================================================
# REVISED COMPREHENSIVE TOR CONFIGURATION (DEBIAN 13)
# ==========================================================

## 1. 基础运行设置
DataDirectory /var/lib/tor
User debian-tor
Log notice file /var/log/tor/notices.log

## 2. 私有 obfs4 网桥设置 (配合前置代理)
BridgeRelay 1
# 内部通信端口，必须与 obfs4 监听端口不同
ORPort 127.0.0.1:9001
# obfs4 监听在本地 10080，供你的 Sing-box/SS 等前置代理转发
ServerTransportPlugin obfs4 exec /usr/bin/obfs4proxy
ServerTransportListenAddr obfs4 127.0.0.1:10080
# 设为 0 以保持网桥私密，不向官方目录发布 
PublishServerDescriptor 0

## 3. 本地客户端 Proxy 设置 (Socks5)
# 供 VPS 本地应用或通过 SSH 隧道使用
SocksPort 127.0.0.1:9050
DNSPort 127.0.0.1:9053
AutomapHostsOnResolve 1
AutomapHostsSuffixes .onion,.exit

## 4. 带宽与限制 (保留你的优化项目) 
RelayBandwidthRate 5 MBytes
RelayBandwidthBurst 10 MBytes

## 5. 安全与出口策略 
ExitPolicy reject *:*
IPv6Exit 0

## 6. 路由优化与地理限制 (保留你的客制化) 
ExcludeNodes {CN},{HK},{MO},{??}
ExitNodes {CH},{NO},{NL},{SE},{DK},{DE},{ES}
StrictNodes 1
EOF
```

四、重启 Tor

```bash
systemctl restart tor
```

五、确认端口监听

```bash
ss -lntup | grep -E '9050|9053'
```
应看到：
```
127.0.0.1:9050
127.0.0.1:9053
```

六、测试 Tor 是否正常
测试 TCP：
```bash
curl --socks5-hostname 127.0.0.1:9050 https://check.torproject.org/api/ip
```
返回包含：
{"IsTor":true}


测试 DNS：
```bash
dig @127.0.0.1 -p 9053 example.com A
```

七、日志排错

```bash
journalctl -u tor -f
```
成功标志：
```bash
Bootstrapped 100% (done): Done
```

八、最终结构

```bash
sing-box → Tor 用户 → 127.0.0.1:9050
sing-box DNS → 127.0.0.1:9053
```

九、常见错误

1. 不要监听 0.0.0.0
2. 不要让 UDP 走 9050
3. 确保 Tor 已 bootstrap 完成
十、确保 /var/lib/tor 目录的归属权是 debian-tor

```bash
chown -R debian-tor:debian-tor /var/lib/tor
```
十、提取网桥
```bash
FINGERPRINT=$(cat /var/lib/tor/fingerprint | awk '{print $2}')
CERT=$(grep -oP 'cert=\K\S+' /var/lib/tor/pt_state/obfs4_bridgeline.txt)
echo "Fingerprint is: $FINGERPRINT"
echo "Cert is: $CERT"
```
组合网桥信息
```bash
obfs4 <你的VPS公网IP>:<前置代理监听的公网端口> <你的FINGERPRINT> cert=<你的CERT> iat-mode=0
```
完成


# Debian 13 VPS：sing-box Reality 入站复用 Tor SocksPort 与私人网桥成功教程

本文只记录最终成功有效的方案。文中需要复制粘贴的命令和配置块，统一使用 ```bash / ```json / ```yaml / ``` 作为代码块边界，避免 Markdown 嵌套时产生歧义。

---

## 0. 最终目标

最终实现：

```
客户端 / Mihomo
  → Reality 443
    → VPS sing-box
      ├─ Direct 用户
      │   ├─ 普通 TCP 直连
      │   ├─ DNS 走 127.0.0.1:53
      │   └─ 访问 VPS_PUBLIC_IP:10080 时 override 到 127.0.0.1:10080 私人网桥
      │
      └─ Tor 用户
          ├─ TCP 走 Tor SocksPort 127.0.0.1:9050
          ├─ DNS 走 Tor DNSPort 127.0.0.1:9053
          └─ 普通 UDP 直接 reject
```

---

## 1. Tor 端配置

编辑 Tor 配置：

```bash
nano /etc/tor/torrc
```

确认至少存在：

```
SocksPort 127.0.0.1:9050
DNSPort 127.0.0.1:9053
```

如果需要 .onion 通过 DNSPort 映射虚拟地址，可额外加入：

```
AutomapHostsOnResolve 1
AutomapHostsSuffixes .onion,.exit
```

重启 Tor：

```bash
systemctl restart tor
```

检查 Tor 端口是否监听：

```bash
ss -lntup | grep -E '9050|9053'
```

理想结果应看到：

```
127.0.0.1:9050
127.0.0.1:9053
```

---

## 2. VPS 端 sing-box 完整配置模板

替换以下占位符：

```
DIRECT_UUID
TOR_UUID
REALITY_PRIVATE_KEY
REALITY_SHORT_ID
VPS_PUBLIC_IP
```

注意：

```
VPS_PUBLIC_IP/32
```

需要替换成你的真实 VPS 公网 IPv4，例如：

```
1.2.3.4/32
```

完整配置如下：

```json
{
  "log": {
    "level": "info",
    "timestamp": true
  },
  "dns": {
    "servers": [
      {
        "tag": "dns-local",
        "type": "udp",
        "server": "127.0.0.1",
        "server_port": 53
      },
      {
        "tag": "dns-tor",
        "type": "udp",
        "server": "127.0.0.1",
        "server_port": 9053
      }
    ],
    "rules": [
      {
        "inbound": [
          "reality-in"
        ],
        "auth_user": [
          "Direct"
        ],
        "action": "route",
        "server": "dns-local",
        "disable_cache": true
      },
      {
        "inbound": [
          "reality-in"
        ],
        "auth_user": [
          "Tor"
        ],
        "action": "route",
        "server": "dns-tor",
        "disable_cache": true
      }
    ],
    "final": "dns-local"
  },
  "inbounds": [
    {
      "type": "vless",
      "tag": "reality-in",
      "listen": "::",
      "listen_port": 443,
      "users": [
        {
          "name": "Direct",
          "uuid": "DIRECT_UUID",
          "flow": "xtls-rprx-vision"
        },
        {
          "name": "Tor",
          "uuid": "TOR_UUID",
          "flow": "xtls-rprx-vision"
        }
      ],
      "tls": {
        "enabled": true,
        "server_name": "developer.mozilla.org",
        "reality": {
          "enabled": true,
          "handshake": {
            "server": "developer.mozilla.org",
            "server_port": 443
          },
          "private_key": "REALITY_PRIVATE_KEY",
          "short_id": [
            "REALITY_SHORT_ID"
          ],
          "max_time_difference": "1m"
        }
      }
    }
  ],
  "outbounds": [
    {
      "type": "direct",
      "tag": "direct"
    },
    {
      "type": "socks",
      "tag": "tor-socks-out",
      "server": "127.0.0.1",
      "server_port": 9050,
      "version": "5",
      "network": "tcp"
    }
  ],
  "route": {
    "default_domain_resolver": {
      "server": "dns-local"
    },
    "rules": [
      {
        "inbound": [
          "reality-in"
        ],
        "auth_user": [
          "Direct"
        ],
        "protocol": "dns",
        "action": "hijack-dns"
      },
      {
        "inbound": [
          "reality-in"
        ],
        "auth_user": [
          "Tor"
        ],
        "protocol": "dns",
        "action": "hijack-dns"
      },
      {
        "inbound": [
          "reality-in"
        ],
        "auth_user": [
          "Direct"
        ],
        "ip_cidr": [
          "VPS_PUBLIC_IP/32"
        ],
        "port": [
          10080
        ],
        "action": "route",
        "outbound": "direct",
        "override_address": "127.0.0.1",
        "override_port": 10080
      },
      {
        "inbound": [
          "reality-in"
        ],
        "auth_user": [
          "Tor"
        ],
        "network": "udp",
        "action": "reject"
      },
      {
        "inbound": [
          "reality-in"
        ],
        "auth_user": [
          "Tor"
        ],
        "network": "tcp",
        "action": "route",
        "outbound": "tor-socks-out"
      },
      {
        "inbound": [
          "reality-in"
        ],
        "auth_user": [
          "Direct"
        ],
        "action": "route",
        "outbound": "direct"
      }
    ],
    "final": "direct"
  }
}
```

---

## 3. VPS 端 sing-box 配置逻辑

Direct 用户：

```
普通 TCP
  → direct

DNS 请求
  → hijack-dns
    → dns-local
      → 127.0.0.1:53

访问 VPS_PUBLIC_IP:10080
  → override_address 127.0.0.1
  → override_port 10080
  → 私人网桥
```

Tor 用户：

```
普通 TCP
  → tor-socks-out
    → 127.0.0.1:9050
      → Tor SocksPort
        → Tor 网络

DNS 请求
  → hijack-dns
    → dns-tor
      → 127.0.0.1:9053
        → Tor DNSPort

普通 UDP
  → reject
```

关键点：

```
Tor SocksPort 9050 只承接 TCP。
Tor DNSPort 9053 只承接 DNS。
Tor 用户普通 UDP 必须拒绝，避免 QUIC / UDP 流量导致超时或泄露。
```

---

## 4. 检查并重启 sing-box

检查配置：

```bash
sing-box check -c /etc/sing-box/config.json
```

格式化配置：

```bash
sing-box format -w -c /etc/sing-box/config.json
```

重启服务：

```bash
systemctl restart sing-box
systemctl restart tor
```

确认监听端口：

```bash
ss -lntup | grep -E '9050|9053|:53|:443|10080'
```

---

## 5. 验证 Tor 服务本身是否正常

测试 Tor SocksPort：

```bash
curl --socks5-hostname 127.0.0.1:9050 https://check.torproject.org/api/ip
```

正常结果应包含：

```json
{"IsTor":true}
```

测试 Tor DNSPort：

```bash
dig @127.0.0.1 -p 9053 example.com A
```

测试本地普通 DNS：

```bash
dig @127.0.0.1 -p 53 example.com A
```

---

## 6. 验证 sing-box 是否真的把 Tor 用户送到了 Tor SocksPort

在 VPS 上开一个窗口监听 9050：

```bash
tcpdump -ni lo tcp port 9050
```

然后客户端使用 Tor 这个 Reality 用户访问：

```
https://check.torproject.org/api/ip
```

如果能访问，并且 tcpdump 看到 9050 上有流量，说明链路成功：

```
客户端
  → Reality Tor 用户
    → VPS sing-box
      → 127.0.0.1:9050
        → Tor 网络
```

---

## 7. Mihomo TUN 全局模式关键设置

如果客户端使用 Mihomo TUN + 全局模式，Tor Reality 节点不能当成全协议 VPN 使用。

Tor Reality 节点需要关闭 UDP：

```yaml
proxies:
  - name: "Tor-Reality"
    type: vless
    server: VPS_PUBLIC_IP
    port: 443
    uuid: TOR_UUID
    flow: xtls-rprx-vision
    network: tcp
    tls: true
    servername: developer.mozilla.org
    client-fingerprint: chrome
    reality-opts:
      public-key: REALITY_PUBLIC_KEY
      short-id: REALITY_SHORT_ID
    encryption: ""
    udp: false
    smux:
      enabled: false
```

不要配置：

```yaml
packet-encoding: xudp
```

在 Mihomo 规则中，本地拒绝 UDP，尤其是 UDP/443 QUIC：

```yaml
rules:
  - AND,((NETWORK,UDP),(DST-PORT,443)),REJECT
  - NETWORK,UDP,REJECT
  - MATCH,Tor-Reality
```

含义：

```
UDP/443 QUIC
  → 本地拒绝，让浏览器回落 TCP

其他普通 UDP
  → 本地拒绝

TCP
  → Tor-Reality
    → VPS sing-box
      → Tor SocksPort
```

---

## 8. Mihomo DNS 建议

TUN 模式下，DNS 往往会先被 Mihomo 自己处理，所以建议使用 fake-ip，让 Mihomo 保留域名映射，再把 TCP 请求交给 Reality 节点。

示例：

```yaml
dns:
  enable: true
  ipv6: false
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  nameserver:
    - system
  proxy-server-nameserver:
    - 223.5.5.5
    - 119.29.29.29
```

核心目标：

```
Mihomo 返回 fake-ip
Mihomo 内部保留 域名 ↔ fake-ip 映射
TCP 请求进入 Tor-Reality
VPS sing-box 把 TCP 送入 Tor SocksPort
由 Tor 完成出口访问
```

---

## 9. 最终成功链路

Mihomo TUN 全局链路：

```
Mihomo TUN 全局
  → 本地拒绝 UDP / QUIC
  → TCP 进入 Tor-Reality
    → Reality 443
      → VPS sing-box reality-in
        → auth_user = Tor
          → TCP route 到 tor-socks-out
            → 127.0.0.1:9050
              → Tor 网络
```

DNS 链路：

```
Direct 用户 DNS
  → VPS sing-box hijack-dns
    → dns-local
      → 127.0.0.1:53

Tor 用户 DNS
  → VPS sing-box hijack-dns
    → dns-tor
      → 127.0.0.1:9053
```

私人网桥链路：

```
Direct 用户访问 VPS_PUBLIC_IP:10080
  → VPS sing-box 匹配 ip_cidr + port
    → override_address 127.0.0.1
    → override_port 10080
      → 本机私人网桥
```

---

## 10. 排错顺序

如果以后又不通，按这个顺序排查：

```bash
systemctl status tor --no-pager
ss -lntup | grep -E '9050|9053|:53|:443|10080'
curl --socks5-hostname 127.0.0.1:9050 https://check.torproject.org/api/ip
dig @127.0.0.1 -p 9053 example.com A
sing-box check -c /etc/sing-box/config.json
journalctl -u sing-box -f
tcpdump -ni lo tcp port 9050
```

判断标准：

```
1. curl --socks5-hostname 127.0.0.1:9050 能返回 IsTor:true
2. dig @127.0.0.1 -p 9053 能解析域名
3. 客户端走 Tor-Reality 时，VPS lo 上能看到 9050 TCP 流量
4. Mihomo 端 UDP 已关闭，并且规则里拒绝 UDP
```

满足这四点，整套链路就是正常的。
