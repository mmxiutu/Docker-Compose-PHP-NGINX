有时候我们需要创建php+nginx环境，不需要额外的套件如redis,mysql等，这时候选择docker环境是非常便捷的

首先需要安装docker和docker-compose

使用阿里云仓库快速安装Docker环境
```
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
```
从github下载compose，下载地址：https://github.com/docker/compose/releases/
下载好之后，将文件移动到系统环境目录，并赋予执行权限
```
mv docker-compose /usr/bin/docker-compose
chmod +x /usr/bin/docker-compose
```
docker环境安装好之后，确保我们可以正常从中央仓库拉取到镜像【可选】
```
docker pull php:7.4.30-fpm-buster
docker pull nginx:1.22.0
```
然后创建必要的目录和配置文件
```
mkdir -p ./nginx/conf/vhost
mkdir -p ./nginx/html/default
mkdir -p ./nginx/logs
mkdir -p ./php
```
文件结构如下
```
arm-web    
├── nginx
│       └── conf
│           └── vhost
│               └── default.conf
│       └── html
│           └── default
│               └── index.html
│               └── phpinfo.php
│       └── logs
├── php
```
其中default.conf文件内容比较重要
```
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;
    #access_log  /var/log/nginx/host.access.log  main;
    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm index.php;
    }
    #error_page  404              /404.html;
    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}
    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    
    location ~ \.php$ {
        root           html;
        fastcgi_pass   arm-phpfpm:9000;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  /var/www/html$fastcgi_script_name;
        fastcgi_param  SCRIPT_NAME $fastcgi_script_name;
        fastcgi_param  PATH_INFO $fastcgi_path_info;
        include        fastcgi_params;
        fastcgi_split_path_info ^(.+\.php)(.*)$;
        fastcgi_param  PATH_TRANSLATED     $document_root$fastcgi_path_info;
    }
    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
}
```
注意配置文件的3个地方：1默认的index文件；2fastcgi_pass指定的ip或者主机名；3fastcgi_param SCRIPT_FILENAME。如果配置错误就会报错404。

如果有php特殊配置，将PHP配置文件命名为php.ini，放入php目录即可。

接下来创建docker-compose.yml文件如下，本分文件是arm下测试的，所以命名以arm开头，可以自行更改
```
version : '3.8'
services:
  arm-phpfpm:
    container_name: arm-phpfpm
    image: php:7.4.30-fpm-buster
    restart: unless-stopped
    expose:
      - "9000"
    volumes:
      - "./php:/usr/local/etc/php"
      - "./nginx/html/default:/var/www/html"
    networks:
      - arm-web-net
  arm-nginx:
    container_name: arm-nginx
    image: nginx:1.22.0
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/html/default:/usr/share/nginx/html
      - ./nginx/logs:/var/log/nginx
      - ./nginx/conf/vhost:/etc/nginx/conf.d
    links:
      - arm-phpfpm
    networks:
      - arm-web-net
networks:
  arm-web-net:

```
最后执行启动命令
```
docker-compose up -d
```
