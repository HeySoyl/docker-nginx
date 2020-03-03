# Docker-Nginx

## 理解Nginx配置文件

```shell
user www-data;
worker_processes auto;
pid /run/nginx.pid;

events {
    worker_connections 768;
    # multi_accept on;
}

http {

    ##
    # Basic Settings
    ##

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    ##
    # SSL Settings
    ##

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
    ssl_prefer_server_ciphers on;

    ##
    # Logging Settings
    ##

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    ##
    # Gzip Settings
    ##

    gzip on;
    gzip_disable "msie6";

    ##
    # Virtual Host Configs
    ##

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

类似events {}和http {}这样的形式，它们在配置文件中叫做块配置项。块配置项可以带参数，也可以嵌套，这取决于提供对应功能的Nginx模块的需要，并且嵌套的内层块会继承外层块的配置。

最后，如果我们要临时关闭某个配置，可以使用#把它注释掉就好了。理解了配置文件的结构之后，这里，有两部分内容是我们要关注的，因为稍后，我们要对其进行修改。

一部分是Nginx的访问和错误日志文件：

```
access_log /var/log/nginx/access.log;
error_log /var/log/nginx/error.log;
```

让Nginx直接把日志保存在容器里是非常不便于查看的，稍后，我们要对这两个路径进行重定向。

另一部分，是底部的include：

```
include /etc/nginx/sites-enabled/*;
```

这和C语言中用#include包含头文件的含义是类似的

## 默认的default配置
默认的default：

```
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    root /var/www/html;

    index index.html index.htm index.nginx-debian.html;

    server_name _;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

同样，为了简洁，我去掉了所有注释的部分。可以看到，里面只有一个server块，每一个server块，都表示一个虚拟的Web服务器，对这个服务器的所有配置，都应该写在server块的内部。其中：

1. listen：表示Nginx服务监听端口的方式，如果我们只填写80，就表示在该服务器上所有IP v4和v6地址上，监听80端口。另外，后面的default_server表示这是Nginx的默认站点，当一个请求无法匹配所有的主机域名时，Nginx就会使用默认的主机配置；
2. root：用于定义资源文件的根目录，默认情况下，Nginx会基于/var/www/html查找要访问的文件；
3. index：表示当请求的URL为/时，访问的文件。当指定多个文件时，Nginx就会从左向右依次找到第一个可以访问的文件并返回；
4. server_name：用于设置服务器的域名，由于我们暂时还在本地开发，因此这里设置成_，表示匹配任何域名；
5. location：可以看到，它也是一个块配置项，用于匹配请求中的URI。这里的含义就是，当用户请求/的时候，执行块内的配置。这里，我们使用了try_files命令，在这个命令的参数里，形如$uri这样的东西，是Nginx或者Nginx模块提供的变量。这里，$uri是Nginx核心HTTP模块提供给我们使用的变量，含义就是请求的完整URI，但是不带任何参数。例如：/a/b/c.jpg。当我们请求这样的资源的时候，就直接尝试查找这个文件，如果第一个参数不存在，就尝试第二个参数，直到最后，我们可以用=404这样的形式，返回一个HTTP Status Code；

### 修改默认default

```
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    root /var/www/html;

    index index.html index.htm index.nginx-debian.html;

    server_name _;

    try_files $uri @proxy;
    location @proxy {
        proxy_pass http://vapor:8080;
        proxy_pass_header Server;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout 5s;
        proxy_read_timeout 10s;
    }
}
```

1. proxy_pass vapor:8080;：proxy_pass命令让Nginx把当前请求反向代理到参数指定的服务器上，这里我们指定的内容是vapor:8080。那么，这个vapor是什么呢？它应该是运行着Vapor服务的服务器的主机名。现在，把它当成是一个替代符就好了。等我们创建并运行了Vapor容器之后，再来解释它；
2. proxy_pass_header Server;：默认情况下，Nginx在把上游服务器的响应转发给客户端的时候，并不会带有一些HTTP头部的字段，例如：Server / Date等。但我们可以通过proxy_pass_header命令，设置允许转发哪些字段。当我们转发了Server字段之后，客户端就会知道实际处理请求的服务器；
3. proxy_set_header Host $host;：由于Nginx作为反向代理的时候，是不会转发请求中的Host头信息的，我们使用了proxy_set_header命令把客户端的Host头信息转发给了上游服务器。这里$host是Nginx HTTP模块提供的变量；
4. X-Real-IP / X-Forwarded-For：这两个字段，前者表示发起请求的原始客户端IP地址；后者用于记录请求被代理的过程里，途径的所有中介服务器的IP地址。稍后，我们会看到这些字段的值，这里就不再多说了，大家知道这些字段的含义就好了；
5. proxy_connect_timeout：设置Nginx和上游服务器连接的超时时间，默认是60秒，我们改成了5秒；
6. proxy_read_timeout：设置Nginx从上游服务器获取响应的超时时间，默认是60秒，我们改成了10秒；

### Nginx镜像

```
FROM ubuntu:18.04

MAINTAINER Mars

RUN apt-get update && apt-get install nginx -y \
        && apt-get clean \
        && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* \
        && echo "daemon off;" >> /etc/nginx/nginx.conf

ADD default /etc/nginx/sites-available/default

RUN ln -sf /dev/stdout /var/log/nginx/access.log && \
    ln -sf /dev/stderr /var/log/nginx/error.log

CMD ["nginx"]
```

完成后，我们只要重新构建镜像就好了：docker build -t soyl/nginx:0.1.2 .
