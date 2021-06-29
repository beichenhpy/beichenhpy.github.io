# 申请Https证书记录
前言💡：一直想把域名搞一下https，以前一直没时间，现在有时间搞一下

## 一、选一家靠谱简单的免费证书发放代理
经过推荐选择了[OHTTPS](https://ohttps.com/)  
优点：
- 邮箱注册
- 教程简单
- 提供DNS-01方式解析

## 二、按照教程配置
1. 在DNS解析服务中，设置对应的CNAME解析
- 主机记录：_acme-challenge.beichenhpy.cn
- 记录值：_acme-challenge.ny5jx0le1zr7m6pg.ohttps.com
2. 等待生成证书
3. 如果你只用CNAME解析(如github page)去访问的话，那么就结束了

## 三、想用nginx进行访问配置
1. 将[dashboard](https://ohttps.com/monitor/certificates?search=&offset=0&limit=10) 中的三个证书文件下载下来，记得将证书文件后缀修改为`.pem` 
   将其放在`/etc/nginx/cert/`目录下
   - 证书文件：`/etc/nginx/cert/cert.pem`
   - 私钥：`/etc/nginx/cert/private.key`
   - 中间文件：`/etc/nginx/cert/fullchain.cer`
2. 配置文件
提供了文件生成网站：[ssl-conf](https://ssl-config.ohttps.com/#server=nginx)
将配置文件放在`/etc/nginx/conf.d/ssh.conf`  
```
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    ssl_certificate /etc/nginx/cert/cert.pem;
    ssl_certificate_key /etc/nginx/cert/private.key;
    ssl_session_timeout 1d;
    ssl_session_cache shared:MozSSL:10m;  # about 40000 sessions
    ssl_session_tickets off;

    # curl https://ssl-config.ohttps.com/ffdhe2048.txt > /etc/nginx/cert/dhparam
    #ssl_dhparam /path/to/dhparam;

    # intermediate configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # HSTS (ngx_http_headers_module is required) (63072000 seconds)
    add_header Strict-Transport-Security "max-age=63072000" always;

    # OCSP stapling
    ssl_stapling on;
    ssl_stapling_verify on;

    # verify chain of trust of OCSP response using Root CA and Intermediate certs
    ssl_trusted_certificate /etc/nginx/cert/fullchain.cer;

    # replace with the IP address of your resolver
    resolver 127.0.0.1;
}
```