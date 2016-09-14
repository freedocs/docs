通过haproxy对流量类型判断，监听同一个端口(80)并根据流量类型选择不同的backend. 由于 `https` 是加密流量，且 `wss` 不能被很好的支持，所以不建议使用 443 端口配置多监听。

## 1\. nginx http 监听到 127.0.0.1

```shell
listen 127.0.0.1:80;
```

## 2\. 安装 haproxy 1.5

```shell
apt-get install software-properties-common
add-apt-repository ppa:vbernat/haproxy-1.5
apt-get update
apt-get install haproxy
```

## 2\. haproxy 配置参数

`vi /etc/haproxy/haproxy.cfg`

```shell
global
    ulimit-n  51200
    log /dev/log    local0 info
    log /dev/log    local1 notice
    chroot /var/lib/haproxy

defaults 
    log global 
    mode tcp
    option dontlognull
    timeout connect 1000 
    timeout client 150000
    timeout server 150000

listen multi-server
    bind x.x.x.x:80
    tcp-request inspect-delay 2s 
    tcp-request content accept if HTTP 
    #acl is_ssl req_ssl_ver 2:3.1

    use_backend http if HTTP 
    use_backend ss if !HTTP

backend http 
    mode http
    option forwardfor header Client-IP 
    option http-server-close 
    server nginx 127.0.0.1:80

backend ss 
    server server-ss :8080 maxconn 20480
```

`x.x.x.x` 为服务器公网ip， 当请求为 `http` 时会转到 nginx 监听的 `127.0.0.1:80` 建立连接，否则会连接到本地的 `8080` 端口
配置完成后 `haproxy` 会打log到 `/var/log/haproxy.log`

