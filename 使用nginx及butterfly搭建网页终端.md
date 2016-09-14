[butterfly](https://github.com/yohanakh/butterfly) 是一个优秀的 网页终端 (web terminal)，可以通过 nginx 构建一个网址为 `https://example.com/butterfly`的网页管理终端，同时使用 `http auth` 认证保证安全。

`nginx` 需要 [ngx_http_substitutions_filter_module](https://github.com/yaoweibin/ngx_http_substitutions_filter_module) 模块的支持，
	
**安装butterfly及依赖**

```bash
pip install butterfly
apt-get purge nginx nginx-full
apt-get install nginx-common libxslt1-dev libgd-dev libgeoip-dev libpcre3-dev git
```

**获取nginx代码**

```bash
# Create temporary work area
cd
mkdir nginx
cd nginx

# Download and extract nginx
wget http://nginx.org/download/nginx-1.9.5.tar.gz
tar xf nginx-1.9.5.tar.gz

# Download and extract OpenSSL
wget https://www.openssl.org/source/openssl-1.0.2d.tar.gz
tar xf openssl-1.0.2d.tar.gz

# Download and extract gzip
wget http://zlib.net/zlib-1.2.8.tar.gz
tar xf zlib-1.2.8.tar.gz

# Delete downloads
rm *.tar.gz

# Download ngx_http_substitutions_filter_module
git clone https://github.com/yaoweibin/ngx_http_substitutions_filter_module
```

**安装编译 nginx**

```bash
cd nginx-1.9.5

./configure \
--with-cc-opt='-g -O2 -fstack-protector --param=ssp-buffer-size=4 -Wformat -Werror=format-security -D_FORTIFY_SOURCE=2' \
--with-ld-opt='-Wl,-Bsymbolic-functions -Wl,-z,relro' \
--sbin-path=/usr/sbin/nginx \
--prefix=/usr/share/nginx \
--conf-path=/etc/nginx/nginx.conf \
--http-log-path=/var/log/nginx/access.log \
--error-log-path=/var/log/nginx/error.log \
--lock-path=/var/lock/nginx.lock \
--pid-path=/run/nginx.pid \
--http-client-body-temp-path=/var/lib/nginx/body \
--http-fastcgi-temp-path=/var/lib/nginx/fastcgi \
--http-proxy-temp-path=/var/lib/nginx/proxy \
--http-scgi-temp-path=/var/lib/nginx/scgi \
--http-uwsgi-temp-path=/var/lib/nginx/uwsgi \
--with-debug \
--with-pcre-jit \
--with-ipv6 \
--with-http_ssl_module \
--with-http_stub_status_module \
--with-http_realip_module \
--with-http_addition_module \
--with-http_dav_module \
--with-http_geoip_module \
--with-http_gzip_static_module \
--with-http_image_filter_module \
--with-http_v2_module \
--with-http_sub_module \
--with-http_xslt_module \
--with-mail \
--with-mail_ssl_module \
--with-http_sub_module \
--with-zlib=../zlib-1.2.8 \
--with-openssl=../openssl-1.0.2d \
--add-module=../ngx_http_substitutions_filter_module

make 
make install
```

**nginx配置**

注意修改 `example.com` 为你的域名

```bash
server {
    listen       80;
    listen       443 ssl;
    server_name  example.com;
    ssl_certificate certs/example.com.chained.crt;
    ssl_certificate_key certs/example.com.key;

    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    charset utf-8;

    access_log  /var/log/nginx/$host.access.log;

    client_max_body_size 20M;

    root   /var/www/;
    index  index.html index.htm index.php;

    if ($ssl_protocol = "") {
        return 301 https://$http_host$request_uri;
    }

    location / {
        try_files $uri $uri/ /index.php?q=$uri&$args;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    location /butterfly {
        auth_basic "Authentication required";
        auth_basic_user_file /etc/nginx/.dlpasswd;

        rewrite ^/butterfly/?(.*) /$1 break;
        proxy_pass        http://127.0.0.1:57575;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        
        proxy_connect_timeout 7d;                                                                                                              
        proxy_send_timeout 7d;                                                                                                                 
        proxy_read_timeout 7d;  

        subs_filter_types text/css text/xml application/javascript;
        subs_filter /style.css '/butterfly/style.css';
        subs_filter /static '/butterfly/static';
        subs_filter /ws '/butterfly/ws';
        subs_filter location.pathname '"/"';
    }
}
```

其中 `/etc/nginx/.dlpasswd` 为htpasswd生成
	
```bash
htpasswd -c /etc/nginx/.dlpasswd xxx
```

**supervisor配置**

注意修改 `user=root` 为可以密码登陆的用户

```bash
vi /etc/supervisor/conf.d/butterfly.conf
```

```ini
[program:butterfly]
command=butterfly.server.py --unsecure --login=false --host=127.0.0.1
autorestart=true
user=root
```

启动 butterfly

```bash
service supervisor restart
```

**检查是否成功**

打开网页访问 `https://example.com/butterfly` 使用网页终端
