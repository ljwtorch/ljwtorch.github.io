# CertBot

## 官方网址

https://certbot.eff.org/

## 申请SSL证书

> 指定Nginx路径方法：
>
> 方式1： certbot --nginx    (当linux有多个版本nginx，会出现找错nginx的配置文件路径)
>
> 方式2： certbot --nginx-server-root  /usr/local/nginx/conf    (指定nginx的配置文件路径)

1. 为单域名申请SSL证书

   ```shell
   # 安装 certbot 以及 certbot nginx 插件
   sudo yum install certbot python2-certbot-nginx -y
   
   # 执行配置，中途会询问你的邮箱，如实填写即可
   sudo certbot --nginx
   
   # 自动续约
   sudo certbot renew --dry-run
   
   # 获得并安装一个证书。
   certbot
   
   # 获得一个证书，但不要安装它。
   certbot certonly
   
   # 你可以用-d指定多个域，通过多次运行Certbot获得并安装不同的证书。
   certbot certonly -d example.com -d www.example.com
   certbot certonly -d app.example.com -d api.example.com
   ```

   > 证书文件会存放在/etc/letsencrypt/archive/http://example.com/目录下，由于采用自动操作，所以Nginx配置certbot会帮我们配置好。

2. 续约

   ```shell
   sudo certbot renew
   
   # 编辑文件
   vim /etc/crontab
   或配置了crontab编辑器的情况下
   crontab -e
   
   # 填入以下内容
   1 1 23 */2 * sudo certbot renew --deploy-hook  "service nginx restart"
   # 注意只有成功renew证书，才会重新启动nginx
   ```

## 重新创建和更新现有证书

您可以使用`certonly`或`run`子命令来请求创建单个新证书，即使您已经拥有具有某些相同域名的现有证书。

如果使用`run`或`certonly`指定已存在的证书名称请求证书，Certbot 会更新现有证书。否则，将创建一个新证书并为其分配指定的名称。

、`--force-renewal`和选项控制 Certbot 在重新创建`--duplicate`与`--expand`现有证书同名的证书时的行为。如果您未指定请求的行为，Certbot 可能会询问您的意图。

`--force-renewal`告诉 Certbot 请求与现有证书具有相同域的新证书。每个域都必须通过`-d`. 如果成功，此证书将与较早的证书一起保存，并且符号链接（“ `live`”引用）将更新为指向新证书。这是更新特定个人证书的有效方法。

`--duplicate`告诉 Certbot 创建一个单独的、不相关的证书，其域与现有证书相同。该证书与之前的证书完全分开保存。大多数用户在正常情况下不需要发出此命令。

`--expand`告诉 Certbot 使用包含所有旧域和一个或多个附加新域的新证书更新现有证书。使用`--expand`选项，使用`-d`选项指定所有现有域和一个或多个新域。

例子：

```
certbot --expand -d existing.com,example.com,newdomain.com
```

如果您愿意，可以像这样单独指定域：

```
certbot --expand -d existing.com -d example.com -d newdomain.com
```

考虑使用`--cert-name`而不是`--expand`，因为它可以更好地控制修改哪个证书，并且允许您删除和添加域。

`--allow-subset-of-names`告诉 Certbot 如果只能获得一些指定的域授权，则继续生成证书。如果证书中指定的某些域不再指向此系统，这可能很有用。

# 内网申请证书

```shell
sudo certbot certonly --manual --preferred-challenges dns -d xxx
# 然后在对应服务商添加TXT记录
# 手动配置nginx
```