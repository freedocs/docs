使用 sniproxy + dnsmasq + nginx 访问互联网

## 优缺点

优点： 客户机简单易用，仅需要修改 dns 即可，甚至局域网内可以不需要做任何修改：主路由修改 dns，则通过网线或 wifi 连接到路由的客户机自动获取到目标 dns。

缺点： 需要目标网站支持 https，对于不支持 https 的网站无效。


## 方案及原理

服务器A: 墙外的 sniproxy 在远端 vps 上监听 443 端口的请求，根据 tls 域名信息来做 https 流量的透明代理。

服务器B: (注意最好是局域网内的机器如路由器或 NAS)

1\. 将所有的 443 端口流量定向到服务器A 443 端口。可以使用 iptables, socat, sniproxy, haproxy 等。 

2\. nginx 将所有的 80 端口请求做 HTTP 302 跳转到 443

3\. dnsmasq 提供一个dns解析服务器，将所有的被污染域名解析到服务器B IP地址，未被污染的域名使用国内的域名服务如 `114.114.114.114` 或 `223.5.5.5` 解析。

客户机C： (如手机、pc等) 修改 dns 设置为服务器 B IP 地址。

**原理**

A: `104.233.233.233` (监听 443) B: `192.168.0.5` (监听 53, 80, 443) C: `192.168.1.10`

1\. 客户机 C 在做 HTTP 请求时，通过 B 提供的 DNS 解析域名，如果是国内网站的域名则直接返回正确的地址，否则返回 服务器B 的IP。如访问 `http://google.com` 时 dns 查询返回了 192.168.0.5

2\. 客户机 C 访问 `http://192.168.0.5:80 (host: google.com)` 后被重定向到 443 端口，客户机 C 继续访问 `https://192.168.0.5:443 (host: google.com)`

3\. 服务器 B 收到 443 端口的请求后，对 tcp 连接定向到 服务器 A 443 端口 即 `https://104.233.233.233:443 (host: google.com)`。

4\. 服务器 A 收到 443 端口的请求后，检查域名 (`google.com`) 并根据域名做透明代理，即 `https://google.com`

***为什么要在本地监听 80 端口？为什么不直接将 dns 解析结果返回 服务器A IP 地址？***

如果直接返回 服务器A IP 地址，客户机 C 在访问 `http://104.233.233.233 (host: google.com)` 时就会导致 tcp reset 而使访问中断。通过在本地局域网监听 80 端口做 302 跳转到 443 后，可以避免这种错误出现。


## 配置示例

服务器A: `/etc/sniproxy.conf`


```
user daemon

pidfile /var/run/sniproxy.pid

error_log {
    syslog daemon
    priority notice
}

listen 104.233.233.233:443 {
    proto tls
    table https_hosts

    access_log {
        filename /var/log/sniproxy/https_access.log
        priority notice
    }
}

table https_hosts {
    .* *:443
}
```

服务器B: 

`/etc/sniproxy.conf`

```
user daemon

pidfile /var/run/sniproxy.pid

error_log {
    syslog daemon
    priority notice
}

listen 443 {
    proto tls
    table https_hosts

    access_log {
        filename /var/log/sniproxy/https_access.log
        priority notice
    }
}

table https_hosts {
    .* 104.233.233.233:443
}
```

`/etc/nginx/sites-enabled/default`

```
server {
    listen 80 default_server;

    server_name _;

    if ($ssl_protocol = "") {
        return 302 https://$http_host$request_uri;
    }
}
```

`/etc/dnsmasq.conf`

```
conf-dir=/etc/dnsmasq.d
listen-address=0.0.0.0
no-resolvserver=8.8.4.4
server=8.8.8.8
address=/#/192.168.0.5
```

`/etc/dnsmasq.d/accelerated-domains.china.conf`

下载 `https://github.com/felixonmars/dnsmasq-china-list/raw/master/accelerated-domains.china.conf` 文件

`/etc/dnsmasq.d/extra.conf`

```
# 手动添加不需要代理的域名列表
server=/baidu.com/114.114.114.114
```
