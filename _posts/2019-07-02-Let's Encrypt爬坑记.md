
想给网站开启https，决定用Let's Encrypt。据说既简单又安全，没想到都是眼泪，于是就有了这篇爬坑记。

打开[Let's Encrypt官网](https://letsencrypt.org/)，跟着Get Started就到了[certbot](https://certbot.eff.org/)。乍一看很人性化，什么nginx，ubuntu18.04 剩下的事情就是跟着步骤一路copy paste和Enter了吧。接下来就发现自己还是太傻太天真了。先选择的是default步骤，报错一看大致意思是DNS provider这边被拒绝了。还有个wildcard方式啊，也发现了linode支持在列。又是一通鼓捣，结果在安装DNS provider插件的时候坑了。通过apt-get安装没有python3-certbot-dns-linode这个包。想了一下直接通过pip安装：pip install cert-dns-linode。安装成功，开心的进行下一步。又坑了，报错大致上提示linode这个插件是0.35.1然后我其它的组件都是0.31.0的版本，版本不兼容！google上查了一下，有个和我一样情况的帖子得到了官方人员的回复：大致意思是说ubuntu的源不是他们官方维护的人家无能为力，并给推荐了一种通用的方式[Running with Docker](https://certbot.eff.org/docs/install.html#id8)。此时我心中有一万头草泥马在崩腾。官方网站上详细描述了安装步骤，出了问题之后轻描淡写的把这个锅甩给了ubuntu。我就想弄个ssl证书，你跟我扯上Docker了。除了“干得漂亮”还能说什么。

吐槽无用，想一下怎么解决吧。cerbot提供的功能无非就是申请好证书，然后再配置nginx（apache）之类的服务器，把这些繁琐的工作工具化。捷径走不通那就一步步的通过源码来解决问题吧。这次终于成功了。先说一下前提，我的服务器是**ubuntu 18.04 LTS（在linoe上）**，nginx是通过apt-get 安装的，事先也需要安装好git，并把你的源更新到最新：sudo apt update && sudo apt upgrade。

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
