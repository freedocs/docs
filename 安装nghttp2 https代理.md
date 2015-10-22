#基本原理

chrome -> nghttpx(https proxy) -> squid -> internet

#参考地址:

[使用 nghttpx 搭建 HTTP/2 代理](https://wzyboy.im/post/1052.html)

[nghttp2](https://github.com/tatsuhiro-t/nghttp2/blob/master/README.rst#requirements)

[spdylay](https://github.com/tatsuhiro-t/spdylay)

#运行环境

    ubuntu 14.04 x86_64

#1\. 安装依赖

    apt-get install make binutils autoconf  automake autotools-dev libtool pkg-config \
    zlib1g-dev libcunit1-dev libssl-dev libxml2-dev libev-dev libevent-dev libjansson-dev \
    libjemalloc-dev cython python3.4-dev
    
#2\. 安装 spdylay

    git clone https://github.com/tatsuhiro-t/spdylay
    
    cd spdylay/
    autoreconf -i
    automake
    autoconf
    ./configure
    make
    make install
    
#3\. 安装 nghttp2

    git clone https://github.com/tatsuhiro-t/nghttp2
    
    cd nghttp2
    autoreconf -i
    automake
    autoconf
	./configure --enable-asio-lib
	make
    make install
    
#4\. 配置 nghttp2

    cp contrib/nghttpx-init /etc/init.d/nghttpx
    mkdir /var/log/nghttpx
    mkdir /etc/nghttpx
    
新建配置文件 `vi /etc/nghttpx/nghttpx.conf`， 添加如下内容
    
    frontend=0.0.0.0,20443
    backend=127.0.0.1,3128
    private-key-file=/etc/nginx/certs/www.example.com.key
    certificate-file=/etc/nginx/certs/www.example.com.crt
    http2-proxy=yes
    
    # set worker，adjust to CPU
    workers=1
    
    # enable client TLS auth
    #verify-client=yes
    #verify-client-cacert=/path/to/client/ca
    
    # not add X-Forwarded-For header
    add-x-forwarded-for=no
    # not add Via header
    no-via=yes
    # not use OCSP server
    no-ocsp=yes
    # set NPN / ALPN order
    npn-list=spdy/3.1,h2
    # only use TLS 1.2
    tls-proto-list=TLSv1.2
    # enable log
    accesslog-file=/var/log/nghttpx/access.log
    accesslog-format=$remote_addr [$time_iso8601] "$request" $status $body_bytes_sent $alpn "$http_user_agent"
    
上边配置会监听20443端口, 并投递到3128端口处理，数据使用tlsv1.2加密传输

#5\. 安装配置 squid

    apt-get update
    apt-get install squid3
    
    cd /etc/squid3/
    mv squid.conf squid.conf.ori
    vi squid.conf
    
添加如下内容

    http_port 127.0.0.1:3128
    #http_access allow localhost
    
    # disable cache and log
    cache deny all
    access_log none
    
    # prefer ipv4
    dns_v4_first on
    # no Via header
    via off
    # delete X-Forwarded-For header
    forwarded_for delete
    
    # http auth
    auth_param basic program /usr/lib/squid3/basic_ncsa_auth /etc/squid3/password
    auth_param basic realm login
    auth_param basic casesensitive on
    auth_param basic credentialsttl 2 hours
    auth_param basic children 5
    acl authenticated proxy_auth REQUIRED
    http_access allow authenticated

    
添加 squid http 认证用户及密码

    htpasswd -c /etc/squid3/password UserName
    
#6\. 运行服务
    
    service squid3 restart
    service nghttpx start
    
如果 nghttpx 没有在后台运行，则需要在 init脚本里增加 daemon 参数。 `ctrl-c` 结束服务，修改 `/etc/init.d/nghttpx` 的
    
    `DAEMON_ARGS="--conf /etc/nghttpx/nghttpx.conf --pid-file=$PIDFILE"`

为

    `DAEMON_ARGS="--conf /etc/nghttpx/nghttpx.conf --pid-file=$PIDFILE --daemon"`
    
#7\. 开机启动服务

    apt-get install sysv-rc-conf
    sysv-rc-conf

使用 `空格` 选中 `nghttpx` 的 `2 3 4 5`， 按 `q` 退出 

#8\. 客户端连接

chrome使用SwitchyOmega插件配置https代理，填写 域名/ip、端口、用户名及密码即可。
