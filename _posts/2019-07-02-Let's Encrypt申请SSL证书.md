---
title: "Let’s Encrypt申请SSL证书"
tags: ["SSL"]
---

想给网站开启https，决定用Let's Encrypt。

先说一下前提，我的服务器是**ubuntu 18.04 LTS（在linode上）**，nginx是通过apt-get 安装的，事先也需要安装好git，并把你的源更新到最新：sudo apt update && sudo apt upgrade。

接下来说一下用certbot生成证书步骤（使用时请将example.com替换成自己的域名）：

 - clone cerbot [repository](https://github.com/certbot/certbot)

   `git clone https://github.com/certbot/certbot /opt/letsencrypt`

   `cd /opt/letsencrypt`

- 运行letsencrypt-auto脚本（进行这一步之前先停止nginx或apache服务器，否则会提示你80或443端口被占用）：

   `sudo -H ./letsencrypt-auto certonly --standalone -d example.com -d www.example.com` 

-  接下来一通Enter即可。

-  此时你网站的域名应该有了，验证一下：

   `sudo ls /etc/letsencrypt/live`

   ```
   example.com
   ```

- 生成证书：

   `./certbot-auto certificates`

    ```
      Found the following certs:
      Certificate Name: example.com
      Domains: example.com www.example.com
      Expiry Date: 2019-09-27 20:49:02+00:00 (VALID: 89 days)
      Certificate Path: /etc/letsencrypt/live/example.com/fullchain.pem
      Private Key Path: /etc/letsencrypt/live/example.com/privkey.pem
    ```

-  Let's Encryptp证书有效期为90天，这一步搞定自动续期：

   `./letsencrypt-auto renew`

   `sudo crontab -e`

   ```bash
   0 0 1 * * /opt/letsencrypt/letsencrypt-auto renew
   ```

-  同理如果你想自动更新cerbot repository：

   `sudo crontab -e`

   ```bash
   0 0 1 * * cd /opt/letsencrypt && git pull
   ```

至此证书的生成就完成了，有了证书配置nginx就简单许多了，接下来说一下nginx配置步骤(可以参考[Configuring HTTPS servers ](http://nginx.org/cn/docs/http/configuring_https_servers.html))：

-  告诉你的server哪有证书（/etc/nginx/sites-enabled/example.conf）：

   ```
   server {
       listen              443 ssl;
       server_name         www.example.com;
       ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
       ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
       ...
   }
   ```

-  http server 优化配置（/etc/nginx/nginx.conf）

   ```
   http {
       ssl_session_cache   shared:SSL:10m;
       ssl_session_timeout 10m;
       ...
   ```

-  重启nginx，享受https带来的快乐：

   `sudo systemctl restart nginx`

最后还可以去[SSL Server Test](https://www.ssllabs.com/ssltest/)验证一下SSL是否已经生效了。
