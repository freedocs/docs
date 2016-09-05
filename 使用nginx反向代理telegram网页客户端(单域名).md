通过反代 telegram api 来实现 telegram 服务的 web 访问。假定域名为 `https://im.example.com`。


##1\. 在 `startssl` 生成 `im` 域名证书

需要反代的域名为 `web.telegram.org`, 如下域名反代为子目录的形式。

`pluto.web` `venus.web` `aurora.web` `vesta.web` `flora.web` `pluto-1.web` `venus-1.web` `aurora-1.web` `vesta-1.web` `flora-1.web`

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

使用方法 `./nginx.sh im.example.com`


##2\. `nginx` 配置


**配置 `web` 程序**

1\. 下载 [webogram](https://github.com/zhukov/webogram/releases) 最新发行版(第一个，不要下载Source code那个tarball)，并解压到服务器 如 `/var/www/im`

2\. 修改 `/var/www/web/js/app.js`，将 `"https://"+l+".web.telegram.org/"` 中的域名修改为 `"https://im.example.com/"+l+"/"` 。这样在访问 `https://im.example.com` 时，调用的 `api` 会返回到 `im.example.com`。 

如 调用的 `https://venus.web.telegram.org/apiw1` 会指向 `https://im.example.com/venus/apiw1`

3\. 下载 `webogram.appcache` 文件到应用目录

```
cd /var/www/im
wget https://web.telegram.org/webogram.appcache
```

4\. 修改应用 `api_id` 及 `api_hash`

在 `https://my.telegram.org` 注册新的应用并填写你的域名，提交后获得 `api_id` 及 `api_hash`

修改 `/var/www/im/js/app.js` 文件，(这里虽然是一堆凌乱得不行的东西，但是可以在nano里使用Ctrl+W搜索)搜索 `Config.App={id:2496,hash:"8da85b0d5bfe62527e5b244c209159c3"`，将其中的 `id` 和 `hash` 修改为你的 `api_id` 和 `api_hash`

5\. 修改文件拥有者为 `http` (archlinux) 或 `www-data` (ubuntu)

`chown -R http:http /var/www/im`


**配置 `api` 代理**

注意替换其中的 `im.example.com` 为你的域名

`/etc/nginx/sites-enabled/im.example.com`

```
server {
    listen       80;
    listen       443 ssl;
    server_name  im.example.com;
    ssl_certificate certs/im.example.com.chained.crt;
    ssl_certificate_key certs/im.example.com.key;

    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    charset utf-8;

    access_log  /var/log/nginx/$host.access.log;

    client_max_body_size 20M;

    root   /var/www/im;
    index  index.html index.htm index.php;

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    location / {
        if ($scheme = http) {
            return 302 https://$http_host$request_uri;
        }
    }

    location ~* .(jpg|jpeg|png|gif|ico|css|js)$ {
        expires 365d;
    }

    # proxy telegram api
    location ~* ^/(pluto|venus|aurora|vesta|flora|pluto-1|venus-1|aurora-1|vesta-1|flora-1)/(.*)$ {
        proxy_buffering off;
        proxy_pass https://$1.web.telegram.org/$2;
    } 
}
```
