---
layout: post
title: Centos7配置免费https证书
---

现在许多网站都支持 HTTPS 了，除了会在地址栏显示绿色锁之外，HTTPS 还有更多的好处。例如开发微信小程序就需要用到 HTTPS，否则请求是无法发送的~

所以我也打算给我的 Centos7 上一个 HTTPS。配置 Https 首先需要一个 SSL 证书，一般这个证书都需要一些权威的 CA 颁发的，如果自己签的证书不会被浏览器认同，也就不会显示绿锁了。

[certbot](https://certbot.eff.org/) 是一个提供自定部署 HTTPS 脚本的网站，它使用了 [Let's Encrypt](https://letsencrypt.org/)的证书。虽然证书的有效期只有一段时间，但是能不断地更新证书的有效期，对于个人使用而言就足够了。

下面是安装的过程和一些总结


----------


首先是幻境依赖的问题，Certbot依赖Python2.7,或者3.4+。除此之外，还需要root权限去写`/etc/letsencrypt`, `/var/log/letsencrypt`, `/var/lib/letsencrypt`这三个文件；绑定80和443端口（如果使用`standalone`插件的话，还须有权限去读写你的 webserver 配置文件（如`apache`或`nginx`)。更多的信息可以参考[cerbot的文档](https://certbot.eff.org/docs/install.html#alternate-installation-methods)。不过这些都不用担心，应为自动脚本会检查你的系统环境，并帮你安装一些必要的依赖。

然后就是你需要先安装你的`webserver`，这里我安装了`nginx`，然后启动`nginx`。


运行下免的命令就可以下载certbot脚本
```
$ wget https://dl.eff.org/certbot-auto
$ chmod a+x ./certbot-auto
$ ./certbot-auto --help
```
这样，如果能看到屏幕打印信息，就说明下载成功了

```
Usage: certbot-auto [OPTIONS]
A self-updating wrapper script for the Certbot ACME client. When run, updates
to both this script and certbot will be downloaded and installed. After
ensuring you have the latest versions installed, certbot will be invoked with
all arguments you have provided.

Help for certbot itself cannot be provided until it is installed.

  --debug                                   attempt experimental installation
  -h, --help                                print this help
  -n, --non-interactive, --noninteractive   run without asking for user input
  --no-bootstrap                            do not install OS dependencies
  --no-self-upgrade                         do not download updates
  --os-packages-only                        install OS dependencies and exit
  --install-only                            install certbot, upgrade if needed, and exit
  -v, --verbose                             provide more output
  -q, --quiet                               provide only update/error output;
                                            implies --non-interactive

All arguments are accepted and forwarded to the Certbot client when run.

```
脚本是通过 https 下载下来的，相对来说比较安全。但如果你想再次检查脚本是否被伪造，可以通过以下方式验证
```
$ wget -N https://dl.eff.org/certbot-auto.asc
$ gpg2 --recv-key A2CFB51FA275A7286234E7B24D17C995CD9775F2
$ gpg2 --trusted-key 4D17C995CD9775F2 --verify certbot-auto.asc certbot-auto
```

然后开始配置 https ,我的环境是centos7 + nginx
```bash
$ ./certbot-auto --nginx
```
安装nginx插件，运行后按照指示安装相应的依赖库
```
$ ./certbot-auto certonly 
```

获取一个证书` ./certbot-auto certonly `
跟着提示步骤选择
```
How would you like to authenticate with the ACME CA?
-------------------------------------------------------------------------------
1: Apache Web Server plugin - Beta (apache)
2: Nginx Web Server plugin - Alpha (nginx)
3: Spin up a temporary webserver (standalone)
4: Place files in webroot directory (webroot)
-------------------------------------------------------------------------------
Select the appropriate number [1-4] then [enter] (press 'c' to cancel): 
```
这里我需要的是nginx，所以我输入2

`Enter email address (used for urgent renewal and security notices) (Enter 'c' to
cancel): `
提示输入邮箱，会通知你关于证书的一些信息
```
Please read the Terms of Service at
https://letsencrypt.org/documents/LE-SA-v1.2-November-15-2017.pdf. You must
agree in order to register with the ACME server at
https://acme-v01.api.letsencrypt.org/directory
```
提示你让你去阅读使用协议，只有同意才能注册

```
Would you be willing to share your email address with the Electronic Frontier
Foundation, a founding partner of the Let's Encrypt project and the non-profit
organization that develops Certbot? We'd like to send you email about EFF and
our work to encrypt the web, protect its users and defend digital rights.
```
提示你能否把你的邮箱地址共享给 Electronic Frontier
Foundation ，这是 Let's Encrypt 的合作伙伴

```
Please enter in your domain name(s) (comma and/or space separated)  (Enter 'c'
to cancel): 
```
这里输入你的域名地址

```
Obtaining a new certificate
Performing the following challenges:
http-01 challenge for jmluang.tk
Using default address 80 for authentication.
Waiting for verification...
Cleaning up challenges

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/jmluang.tk/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/jmluang.tk/privkey.pem
   Your cert will expire on 2018-06-22. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot-auto
   again. To non-interactively renew *all* of your certificates, run
   "certbot-auto renew"
 - Your account credentials have been saved in your Certbot
   configuration directory at /etc/letsencrypt. You should make a
   secure backup of this folder now. This configuration directory will
   also contain certificates and private keys obtained by Certbot so
   making regular backups of this folder is ideal.
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```
到这里证书就生成啦，还提示了一些关键的信息给你知，包括证书存放的位置，还有证书过期的时间以及如何续期证书，以及提醒你应该及时备份你的 /etc/letsencrypt 目录。还有就是如果有用的话可以进行一定的捐赠

然后就是配置nginx了
```
// http 转跳到 https 
server {
        listen       80;
        server_name  jmluang.tk;
	    rewrite ^(.*) https://$server_name$1 permanent;
}

// https 配置
server {
        listen       443 ssl http2;
        #listen       80;
        server_name  jmluang.tk;
        ssl           on;
        ssl_certificate "/etc/letsencrypt/live/jmluang.tk/fullchain.pem";
        ssl_certificate_key "/etc/letsencrypt/live/jmluang.tk/privkey.pem";
#        ssl_session_cache shared:SSL:1m;
#        ssl_session_timeout  10m;
#        ssl_ciphers HIGH:!aNULL:!MD5;
#        ssl_prefer_server_ciphers on;
 
        location / {
                root  /var/www/html;
                index  index.html;
        }
 
}
```
这里`ssl_certificate` 和 `ssl_certificate_key` 的文件路径都在上面提到的 `/etc/letsencrypt`目录中，你只要修改你的域名就可以了。其他的不需要改动，配置文件都是一样的~

最后应用配置到`nginx` ：`nginx -s reload`

访问域名，出现绿色锁就完成啦！

最后到期了记得运行 `certbot-auto renew ` 为你的证书续期哦~ 也可以加入到crontab中自动运行！



> Written with [StackEdit](https://stackedit.io/).
