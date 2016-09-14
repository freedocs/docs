TEWA-500G

地址： 192.168.1.1
ssh： admin:admin ，进入后运行sh

挂载读写权限

```bash
mount -o remount rw /
```

1\. 修改用户名只能使用 `useradmin` 的问题

```bash
cd /webs
vi login.html
```

查找 `disabled`， 将 `value=\"useradmin\" disabled='true'` 改为 `value=\"useradmin\"`

2\. 修改 ssid 必须为 ChinaNet- 开头的问题

vi NW_Basic.html

删除如下行

```js
     var place = str.indexOf("ChinaNet-");
     if(place!=0)
     {
            alert('SSID "' + wlSsid.value + '" ......ChinaNet-..............
            return false;
     }
```

-------------

网页端 `192.168.1.1` 登录 `telecomadmin` `nE7jA%5m` 修改 `ssid`，开启 `pppoe` 自动拨号 	 

3\. 添加 PPPOE 自动拨号，使 wifi 可以上网。

网络-网络设置- 2_INTERNET_B_VID_ 连接模式 路由，输入宽带用户名和密码

保存，应用。

在状态，网络侧信息即可查看 PPPOE IP地址。检查 电视机 IPTV 是否正常。

4\. 修改自动获取的 dns

电信自己的 dns 解析慢，而且有很多问题，如果使用猫的网络，客户端就需要配置dns。所以如果在猫上直接修改dns就可以避免客户端修改的麻烦。

在 ssh 控制台里

```bash
cd /etc
ls -l
resolv.conf -> /var/fyi/sys/dns
```

可以看到 resolv.con 指向了 /var/fyi/sys/dns， 每次拨号成功后会修改 这个文件 到电信默认的 dns

```bash
mv resolv.conf resolv.conf.bak
vi resolv.conf
```
添加阿里和 114 的 dns

```bash
nameserver 223.5.5.5
nameserver 114.114.114.114
```

保存， reboot 重启路由器，拨号后可以看到 `resolv.conf` 仍是我们修改的dns，而 `resolv.conf.bak` 则会发生变化。
