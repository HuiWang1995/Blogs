# debian 11 安装nginx 并配置端口映射

> 问：Debian 11 x64 安装nginx 并配置 443 与80 映射到8080端口

## chatgpt的回答

安装nginx并配置端口映射的步骤如下：

1. 安装nginx

   在Debian 11 x64上安装nginx可以使用以下命令：

   ```
   sudo apt update
   sudo apt install nginx
   ```

   安装完成后，nginx就会自动启动并监听80端口。

2. 配置端口映射

   为了将80和443端口映射到8080端口，需要修改nginx的配置文件`/etc/nginx/sites-available/default`。可以使用以下命令打开该文件：

   ```
   sudo nano /etc/nginx/sites-available/default
   ```

   在该文件中，找到以下两行：

   ```
   listen [::]:80 default_server;
   listen 80 default_server;
   ```

   将它们修改为：

   ```
   listen [::]:8080 default_server;
   listen 8080 default_server;
   ```

   然后，在该文件中再找到以下两行：

   ```
   listen [::]:443 ssl default_server;
   listen 443 ssl default_server;
   ```

   将它们修改为：

   ```
   listen [::]:8443 ssl default_server;
   listen 8443 ssl default_server;
   ```

   保存并关闭该文件。

3. 重启nginx

   配置完成后，需要重启nginx才能使配置生效。可以使用以下命令重启nginx：

   ```
   sudo systemctl restart nginx
   ```

   重启完成后，nginx就会监听8080和8443端口，并将它们映射到80和443端口。

现在，您可以通过访问服务器的IP地址或域名来访问nginx服务器了，例如：

```
http://your_server_ip/
https://your_server_ip/
```

这些请求会被映射到8080或8443端口，并由nginx处理。

## 实际的操作
chatgpt 并不是万能的，存在错误和学习不及时。

1. 实际上，debian系安装的确如此`apt install nginx` 即可，不过生成的配置文件在`/etc/nginx/nginx.conf`。
2. 443端口是https的默认端口，还需要配置证书的，当然，修改配置文件后，重启生效是正确的。

## 最简配置文件
```text
# 为注释，无需删除，不影响
#user www-data;
worker_processes 2;
#pid /run/nginx.pid;
#include /etc/nginx/modules-enabled/*.conf;

events {
        worker_connections 768;
        # multi_accept on;
}

http {
    # HTTP请求转发配置
    server {
        listen 80;
        # 域名
        server_name wangpig.life;
        location / {
            # 代理的IP:端口
            proxy_pass http://wangpig.life:8080;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }

    # HTTPS请求转发配置
    server {
        listen 443 ssl;
        server_name wangpig.life;
        # 申请的证书所在的位置
        ssl_certificate /root/zero-ssl-cert/certificate.crt;
        ssl_certificate_key /root/zero-ssl-cert/private.key;

        location / {
            proxy_pass http://wangpig.life:8080;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}

```

## 遗漏的操作

开通网络规则，或者关闭防火墙。没有关闭防火墙会导致外部无法访问。
而本机通过curl则可请求得到响应。**因为忘记修改防火墙，导致我这个nginx新手以为配置错误，查了好久，是有点菜了。**

在 Debian 11 上，可以使用以下命令来关闭防火墙：

1. 如果你使用的是 UFW 防火墙，可以使用以下命令来关闭它：

```
sudo ufw disable
```

2. 如果你使用的是 iptables 防火墙，可以使用以下命令来清空规则并关闭它：

```
sudo iptables -F
sudo iptables -X
sudo iptables -P INPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
sudo iptables -P OUTPUT ACCEPT
sudo iptables-save > /etc/iptables/rules.v4
sudo systemctl stop iptables
sudo systemctl disable iptables
```

请注意，关闭防火墙会使你的系统更加容易受到攻击，因此建议在必要的情况下才关闭防火墙。如果你需要在特定端口上打开流量，请确保只允许必要的流量通过，并使用其他安全措施来保护你的系统。