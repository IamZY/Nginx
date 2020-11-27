# Nginx

+ nginx.conf

```shell

#user  nobody;
# 全局块：主要会设置一些影响nginx服务器整体运行的配置命令，
#        主要包括配置运行nginx服务器的用户组，允许生成的worker process数量，进程PID 存放路径日志存放路径和类型以及配置文件的引入
worker_processes  1;  # 并发处理的数量

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


#  events 块：影响 Nginx 服务器与用户的网络连接
# 比如 worker_connections 1024; 支持的最大连接数为 1024
events {
    worker_connections  1024;
}

# http 块 还包含两部分： http 全局块 server 块
http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;
    # 负载均衡
    upstream myserver{
        server 192.168.52.128:8080;
        server 192.168.52.128:8081;
    }
    server {
        listen       80;
        server_name  192.168.52.128;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   html;
            # 负载均衡
            proxy_pass http://myserver;
            # proxy_pass http://127.0.0.1:8080;
            index  index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }


    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    server {
       listen       9001;
       server_name  192.168.52.128;

       location ~ /edu/ {
           proxy_pass http://127.0.0.1:8080;
       }
       location ~ /vod/ {
           proxy_pass http://127.0.0.1:8081;
       }

    }


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}

}

```

## Centos6更换阿里yum源

### 首先备份原来的centos官方yum源

> cp /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak

### 获取阿里的yum源覆盖本地官方yum源

> wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-6.repo

### 清理yum缓存，并生成新的缓存

> yum clean all
>
> yum makecache

## 高可用

> yum install keepalived -y

+ keepalived.conf

  ```shell
  ! Configuration File for keepalived
  
  # 全局定义
  global_defs {
     notification_email {
       acassen@firewall.loc
       failover@firewall.loc
       sysadmin@firewall.loc
     }
     notification_email_from Alexandre.Cassen@firewall.loc
     smtp_server 192.168.52.129
     smtp_connect_timeout 30
     router_id LVS_DEVEL
  }
  
  vrrp_script chk_http_port{
    script "/usr/local/src/nginx_check.sh"
    interval 2  # 检测脚本间隔时间
    weight 2
  }
  
  vrrp_instance VI_1 {
      state BACKUP  # 备份服务器上将MASTER 改为 BACKUP
      interface eth2  # 网卡
      virtual_router_id 51  # 主备机的virtual_router_id 必须相同
      priority 90 # 主备机不同的优先级 主机值较大 备份机值较小
      advert_int 1
      authentication {
          auth_type PASS
          auth_pass 1111
      }
      virtual_ipaddress {
          192.168.52.16   // VRRP 虚拟地址
      }
  }
  
  ```

+ service start keepalived

+ nginx_check.sh

  ```shell
  #!/bin/bash
  A=`ps -C nginx ¨Cno-header |wc -l`
  if [ $A -eq 0 ];then
      /usr/local/nginx/sbin/nginx
      sleep 2
      if [ `ps -C nginx --no-header |wc -l` -eq 0 ];then
          killall keepalived
      fi
  fi
  ```

  
