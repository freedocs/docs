环境： ubuntu server 14.04， ssl二级域名证书及私钥放在 `/etc/nginx/certs/www.example.com.crt` 和 `/etc/nginx/certs/www.example.com.key`

**1\. 安装 spdylay 及 nghttp2**

```
cd 
mkdir tmp
cd tmp
wget https://github.com/freedocs/binary/raw/master/http2/nghttp2_1.0.5-DEV-1_amd64.deb
wget https://github.com/freedocs/binary/raw/master/http2/spdylay_1.3.3-DEV-1_amd64.deb

dpkg -i nghttp2_1.0.5-DEV-1_amd64.deb
dpkg -i spdylay_1.3.3-DEV-1_amd64.deb

rm *.deb
```

配置

```
wget https://github.com/freedocs/binary/raw/master/http2/nghttpx-init -O /etc/init.d/nghttpx
chmod +x /etc/init.d/nghttpx
mkdir /var/log/nghttpx/
mkdir /etc/nghttpx
wget https://github.com/freedocs/binary/raw/master/http2/nghttpx.conf.sample -O /etc/nghttpx/nghttpx.conf
```

修改 `/etc/nghttpx/nghttpx.conf` 中的 `www.example.com` 为你的域名


**2\. 安装配置 squid**

```
apt-get update
apt-get install squid3 apache2-utils -y
cd /etc/squid3/
mv squid.conf squid.conf.ori
wget https://github.com/freedocs/binary/raw/master/http2/squid.conf.sample -O squid.conf
```

添加 squid http 认证用户及密码

```
htpasswd -c /etc/squid3/password UserName
```

**3\. 运行服务**

```
service squid3 restart
service nghttpx start
```

**4\. 设置开机启动**

```
update-rc.d nghttpx defaults
```

**一键脚本**

```
#!/bin/bash

echo "Install nghttpx..."

cd 
mkdir tmp
cd tmp
wget https://github.com/freedocs/binary/raw/master/http2/nghttp2_1.0.5-DEV-1_amd64.deb
wget https://github.com/freedocs/binary/raw/master/http2/spdylay_1.3.3-DEV-1_amd64.deb

dpkg -i nghttp2_1.0.5-DEV-1_amd64.deb
dpkg -i spdylay_1.3.3-DEV-1_amd64.deb

rm *.deb

wget https://github.com/freedocs/binary/raw/master/http2/nghttpx-init -O /etc/init.d/nghttpx
chmod +x /etc/init.d/nghttpx
mkdir /var/log/nghttpx/
mkdir /etc/nghttpx
wget https://github.com/freedocs/binary/raw/master/http2/nghttpx.conf.sample -O /etc/nghttpx/nghttpx.conf

read -p "Input domain name: " DOMAIN

sed -i "s/www.example.com/$DOMAIN/g" /etc/nghttpx/nghttpx.conf

echo "Install squid3..."

apt-get update
apt-get install squid3 apache2-utils libev4 libjemalloc1 -y
cd /etc/squid3/
mv squid.conf squid.conf.ori
wget https://github.com/freedocs/binary/raw/master/http2/squid.conf.sample -O squid.conf

read -p "Input http username: " USERNAME

htpasswd -c /etc/squid3/password $USERNAME

echo "start services..."

service squid3 restart
service nghttpx start

update-rc.d nghttpx defaults

echo "done."
```
