## 已弃用，请参考 [使用nginx反向代理telegram网页客户端(单域名)](https://github.com/freedocs/docs/blob/master/%E4%BD%BF%E7%94%A8nginx%E5%8F%8D%E5%90%91%E4%BB%A3%E7%90%86telegram%E7%BD%91%E9%A1%B5%E5%AE%A2%E6%88%B7%E7%AB%AF(%E5%8D%95%E5%9F%9F%E5%90%8D).md)

通过反代 telegram api 来实现 telegram 服务的 web 访问。假定域名为 `https://web.example.com`。


##1\. 在 `startssl` 生成域名证书，例如需要如下几个子域名, 用于替换 `web.telegram.org`


`web` `pluto.web` `venus.web` `aurora.web` `vesta.web` `flora.web` `pluto-1.web` `venus-1.web` `aurora-1.web` `vesta-1.web` `flora-1.web`

配置域名解析。

证书及私钥保存在 `/etc/nginx/certs/` 目录。使用如下脚本 `nginx.sh` 配置证书的编译链。

`/etc/nginx/certs/nginx.sh`

```
#!/bin/bash

#if [ ! -f "ca-sha2.pem" ];then
#    wget http://www.startssl.com/certs/ca-sha2.pem -O ca-sha2.pem
#fi

if [ ! -f "sub.class1.server.sha2.ca.pem" ];then
    wget https://www.startssl.com/certs/class1/sha2/pem/sub.class1.server.sha2.ca.pem -O sub.class1.server.sha2.ca.pem
fi

if [ ! -f "ca-certs.crt" ];then
    cat sub.class1.server.sha2.ca.pem > ca-certs.crt
fi

cat $1.crt ca-certs.crt > $1.chained.crt
```

使用方法 `./nginx.sh pluto.web.example.com`

或者使用如下脚本批量操作

`/etc/nginx/certs/chainall.sh`

```
#!/bin/bash

for file in *.crt;
do
    ./nginx.sh ${file%%.crt*};  
done

```

##2\. `nginx` 配置


**配置 `web` 程序**

1\. 下载 [webogram](https://github.com/zhukov/webogram/releases) 最新发行版，并解压到服务器 如 `/var/www/web`

2\. 修改 `/var/www/web/js/app.js`，将 `"https://"+l+".web.telegram.org/"` 中的域名修改为你自己的域名。这样在访问 `https://web.example.com` 时，调用的 `api` 会返回到 `web.example.com`。 

如 调用的 `https://venus.web.telegram.org/apiw1` 会指向 `https://venus.example.com/apiw1`

3\. 下载 `webogram.appcache` 文件到应用目录

```
cd /var/www/web
wget https://web.telegram.org/webogram.appcache
```

4\. 修改应用 `api_id` 及 `api_hash`

在 `https://my.telegram.org` 注册新的应用并填写你的域名，提交后获得 `api_id` 及 `api_hash`

修改 `/var/www/web/js/app.js` 文件，搜索 `Config.App={id:2496,hash:"8da85b0d5bfe62527e5b244c209159c3"`，将其中的 `id` 和 `hash` 修改为你的 `api_id` 和 `api_hash`

5\. 修改文件拥有者为 `http` (archlinux) 或 `www-data` (ubuntu)

`chown -R http:http /var/www/web`


**配置 `api` 代理**

注意替换其中的 `web.example.com` 为你的域名

`/etc/nginx/sites-enabled/web.example.com`

```
server {
    listen       80;
    listen       443 ssl;
    server_name  web.example.com;
    ssl_certificate certs/web.example.com.chained.crt;
    ssl_certificate_key certs/web.example.com.key;

    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    charset utf-8;

    access_log  /var/log/nginx/$host.access.log;

    client_max_body_size 20M;

    root   /var/www/web;
    index  index.html index.htm index.php;

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    location / {
        if ($scheme = http) {
            return 302 https://$http_host$request_uri;
        }
        
#        proxy_buffering off;
#        proxy_pass https://web.telegram.org;
    }

    location ~* .(jpg|jpeg|png|gif|ico|css|js)$ {
        expires 365d;
    }
}

server {
    listen       80;
    listen       443 ssl;
    server_name  venus.web.example.com;
    ssl_certificate certs/venus.web.example.com.chained.crt;
    ssl_certificate_key certs/venus.web.example.com.key;

    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    if ($scheme = http) {
        return 302 https://$http_host$request_uri;
    }

    location / {
        proxy_buffering off;
        proxy_pass https://venus.web.telegram.org;
    }
}

server {
    listen       80;
    listen       443 ssl;
    server_name  pluto.web.example.com;
    ssl_certificate certs/pluto.web.example.com.chained.crt;
    ssl_certificate_key certs/pluto.web.example.com.key;

    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    if ($scheme = http) {
        return 302 https://$http_host$request_uri;
    }

    location / {
        proxy_buffering off;
        proxy_pass https://pluto.web.telegram.org;
    }
}

server {
    listen       80;
    listen       443 ssl;
    server_name  aurora.web.example.com;
    ssl_certificate certs/aurora.web.example.com.chained.crt;
    ssl_certificate_key certs/aurora.web.example.com.key;

    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    if ($scheme = http) {
        return 302 https://$http_host$request_uri;
    }

    location / {
        proxy_buffering off;
        proxy_pass https://aurora.web.telegram.org;
    }
}

server {
    listen       80;
    listen       443 ssl;
    server_name  vesta.web.example.com;
    ssl_certificate certs/vesta.web.example.com.chained.crt;
    ssl_certificate_key certs/vesta.web.example.com.key;

    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    if ($scheme = http) {
        return 302 https://$http_host$request_uri;
    }

    location / {
        proxy_buffering off;
        proxy_pass https://vesta.web.telegram.org;
    }
}

server {
    listen       80;
    listen       443 ssl;
    server_name  flora.web.example.com;
    ssl_certificate certs/flora.web.example.com.chained.crt;
    ssl_certificate_key certs/flora.web.example.com.key;

    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    if ($scheme = http) {
        return 302 https://$http_host$request_uri;
    }

    location / {
        proxy_buffering off;
        proxy_pass https://flora.web.telegram.org;
    }
}

server {
    listen       80;
    listen       443 ssl;
    server_name  pluto-1.web.example.com;
    ssl_certificate certs/pluto-1.web.example.com.chained.crt;
    ssl_certificate_key certs/pluto-1.web.example.com.key;

    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    if ($scheme = http) {
        return 302 https://$http_host$request_uri;
    }

    location / {
        proxy_buffering off;
        proxy_pass https://pluto-1.web.telegram.org;
    }
}

server {
    listen       80;
    listen       443 ssl;
    server_name  venus-1.web.example.com;
    ssl_certificate certs/venus-1.web.example.com.chained.crt;
    ssl_certificate_key certs/venus-1.web.example.com.key;

    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    if ($scheme = http) {
        return 302 https://$http_host$request_uri;
    }

    location / {
        proxy_buffering off;
        proxy_pass https://venus-1.web.telegram.org;
    }
}

server {
    listen       80;
    listen       443 ssl;
    server_name  aurora-1.web.example.com;
    ssl_certificate certs/aurora-1.web.example.com.chained.crt;
    ssl_certificate_key certs/aurora-1.web.example.com.key;

    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    if ($scheme = http) {
        return 302 https://$http_host$request_uri;
    }

    location / {
        proxy_buffering off;
        proxy_pass https://aurora-1.web.telegram.org;
    }
}

server {
    listen       80;
    listen       443 ssl;
    server_name  vesta-1.web.example.com;
    ssl_certificate certs/vesta-1.web.example.com.chained.crt;
    ssl_certificate_key certs/vesta-1.web.example.com.key;

    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    if ($scheme = http) {
        return 302 https://$http_host$request_uri;
    }

    location / {
        proxy_buffering off;
        proxy_pass https://vesta-1.web.telegram.org;
    }
}

server {
    listen       80;
    listen       443 ssl;
    server_name  flora-1.web.example.com;
    ssl_certificate certs/flora-1.web.example.com.chained.crt;
    ssl_certificate_key certs/flora-1.web.example.com.key;

    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    if ($scheme = http) {
        return 302 https://$http_host$request_uri;
    }

    location / {
        proxy_buffering off;
        proxy_pass https://flora-1.web.telegram.org;
    }
}
```
