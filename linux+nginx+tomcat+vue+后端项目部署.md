##    								 linux+nginx+tomcat+vue+后端项目部署

#### 1.nginx安装

##### 1.先安装依赖

```
yum -y install gcc gcc-c++ autoconf automake
yum -y install PCRE pcre-devel
yum -y install zlib zlib-devel
yum -y install openssl openssl-devel
```

##### 2.安装nginx

```
cd /usr/local/source
wget  http://nginx.org/download/nginx-1.9.0.tar.gz
tar -zxvf nginx-1.6.2.tar.gz
mv nginx-1.6.2 /usr/local/nginx
cd /usr/local/nginx/
```

\#预编译Nginx

```
useradd www;
cd /usr/local/nginx/nginx-1.xx.x
./configure --user=www --group=www --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module
```

\#.configure预编译成功后，执行make命令进行编译

```
make
```

\#make执行成功后，执行make install 正式安装（此时会在nginx文件夹下生成一个sbin目录）

```
make install
```

#运行一下命令

```
/usr/local/nginx/sbin/nginx -t 检查nginx配置文件是否正确
/usr/local/nginx/sbin/nginx 启动nginx
ps -aux | grep nginx 查看nginx进程
/usr/local/nginx/sbin/nginx -s reload 重启nginx
/usr/local/nginx/sbin/nginx -v 查看nginx版本
```

#修改nginx.conf文件(/nginx/conf/nginx.conf,可以先不改，前面运行成功nginx,如果查到版本号，并且访问ip，出现nginx下图表示安装成功,nginx的默认端口是80)

```
 #nginx的默认负载均衡策略upstream
 upstream iot {
     server   192.168.126.128:8080 weight=1 max_fails=2 fail_timeout=30s;
     server   192.168.126.128:8081 weight=1 max_fails=2 fail_timeout=30s;
}

```

![img](https://img-blog.csdn.net/20180413105031812)

#### 2.nginx.conf解释（根据实际情况配置）

#####     1.server

![img](https://img-blog.csdn.net/20161110175807267)

 

```
这段代码是在配置文件中的server中，一个server相当于一个代理服务器，可以配置多个server。

里面几个属性的意思分别是：

listen：代表当前代理服务器的访问端口号，默认是80端口。如果要配置多个server，这里的默认端口需要改变，要不然系统不知道进入哪个代理服务。

server_name：表示代理服务需要转发的地址，默认是localhost。

location：表示匹配客户端发送请求的路径，这里“/”代表所有请求的路径都能匹配,配置代理就是这个,location /表示全部请求都要经过这里。

root：表示请求别匹配到后，会在这个文件夹内寻找相应的文件，root对后面静态资源的处理很重要。

index：如果代理没有指定主页，将默认进入index配置下寻找主页，可以配置多个，第一个主页找不到，访问第二个，以此类推。

error_page：代表发生错误后进入的相关错误页面，下面的location也是处理错误的相关配置。
```

##### 		2.负载均衡配置及解释(放在http里或https里)

​			1)	 ip hash:依据ip分配方式，指定负载均衡器按照基于客户端ip的分配方式，这个方法确保了客户端请求一致发送到相同的服务器，保证了session会话，每次都访问同一个后端的服务器。可以解决session不能跨服务器的问题

```
 #nginx的默认负载均衡策略upstream
 upstream [服务器名称] {
 	 ip hash;
     server   [ip地址]:[端口号];
     server   [ip地址]:[端口号];
     ...
}
```

​			2） 轮询：按时间顺序平均分配到不同的后端服务器

```
#nginx的默认负载均衡策略upstream
 upstream [服务器名称] {
     server   [ip地址]:[端口号];
     server   [ip地址]:[端口号];
     ...
}
```

​			3）weight:权重方式,在轮询的基础上指定轮询的几率

```
#nginx的默认负载均衡策略upstream
 upstream iot {
     server   [ip地址]:[端口号] weight=2;
     server   [ip地址]:[端口号]  weight=6;
     ...
}
```

#####   		3.worker_processes  启动进程数，一般跟cpu的相等

```:
worker_processes  4
linux查询cpu数命令: cat /proc/cpuinfo| grep "physical id"| sort| uniq| wc -l
```

##### 		 4.配置多个location

```
server {
        listen       8989;
        server_name  localhost;
		access_log logs/esb.log;
		error_log logs/esb-error.log;
		#将所有请求转发给 esb 的应用处理
		
		location = /uaapi {
			proxy_buffer_size 128k;
			proxy_buffers 32 32k;
			proxy_busy_buffers_size 128k;
			proxy_pass http://192.168.31.83:8086/uaapi/addressManage/getAddressByDept?dept=5019;
		}
		
		location /uaapi/dc {
			proxy_buffer_size 128k;
			proxy_buffers 32 32k;
			proxy_busy_buffers_size 128k;
			proxy_pass http://127.0.0.1:8086;
		}
		
		location /dd {
			proxy_buffer_size 128k;
			proxy_buffers 32 32k;
			proxy_busy_buffers_size 128k;
			proxy_pass http://127.0.0.1:8086/uaapi/dc/addressManage/getAddressByDept?dept=5019;
		}
    }
```

#####         5.配置多个location(前端)

```
location / {
         root   /data/html/;
         index  index.html index.html;
    }
    location / {
         root   /data/html/;
         index  index.html index.html;
    }
    location /train {
         root   /data/trainning/;
         index  index.html index.html;
    }
    #location如果一个特定的url 要使用别名，不能用root，alias指定的目录是准确的，root是指定目录的上级目录，改动后即可以使用了
```



#### 3.tomcat的安装及配置

1.官网下载压缩包: [](http://tomcat.apache.org/download-80.cgi)

2.复制到目录解压:  tar -zxv -f apache-tomcat-8.5.37.tar.gz 

3.如果要配置多个tomcat可改server.xml:   Server port="8005" 、 Connector port="8080" protocol="HTTP/1.1" 、Connector port="8009" protocol="AJP/1.3"这几个的端口保证每个tomcat的这几端口都不一样

4.端口解释: server prot 是tomcat运行的端口，Connector port="8080" protocol="HTTP/1.1是请求端口，也是客户端发送请求过来的端口，也是我们访问的端口，Connector port="8009" protocol="AJP/1.3"是tomcat的监听端口。

###### tomcat配置多个：

1. 操作/etc/profile 文件: vi /etc/profile

2. ```
   #tomcat1
   export CATALINA_HOME1=/home/tomcat/tomcat8.5_1
   export CATALINA_BASE1=/home/tomcat/tomcat8.5_1
   export TOMCAT_HOME1=/home/tomcat/tomcat8.5_1
   
   #tomcat2
   export CATALINA_HOME2=/home/tomcat/tomcat8.5_2
   export CATALINA_BASE2=/home/tomcat/tomcat8.5_2
   export TOMCAT_HOME2=/home/tomcat/tomcat8.5_2
   
    ...
   ```

  3.改tomcat的 server.xml：上面已经说过了要保证每个端口都不一样即可

  4.改startup.sh 和 shutdown.sh文件这两分别是tomcat运行的文件和停止的文件

```
#对应你上面profile配置的名称,两文件都加上就行了
export CATALINA_BASE=$CATALINA_BASE1
export CATALINA_HOME=$CATALINA_HOME1
export TOMCAT_HOME=TOMCAT_HOME1
```

​	5.运行一下看看是否有错,没错收工!

#### 4.实际项目配置

1.  nginx.conf:

   ```
   
   #user  nobody;
   worker_processes  1; #启动进程数一般和cpu数相等
   
   #error_log  logs/error.log;
   #error_log  logs/error.log  notice;
   #error_log  logs/error.log  info;
   
   #pid        logs/nginx.pid;
   
   
   events {
       worker_connections  1024; #一个worker的最大连接数
   }
   
   
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
       
   	#负载均衡策略，这里的是weight权重概率
       upstream iot {
            server   localhost:8081  weight=1 max_fails=2 fail_timeout=30s;
            server   localhost:8080  weight=1 max_fails=2 fail_timeout=30s;
       }
   
       server {
           listen       80; #监听端口
           server_name  localhost; #地址
   
           #charset koi8-r;
   
           #access_log  logs/host.access.log  main;
   
           location / {		
               root   /sr/vue/iot; #静态资源路径（重要）
               try_files $uri $uri/ /index.html; #防止重定向页面刷新 路由失效
               index  index.html index.html;
   	 		#   proxy_pass http://iot;
           }
   
   
   	location /iot/ {
                   proxy_pass http://iot; #代理地址用了负载均衡和iot上面的负载均衡名一样
            		#下面5个是定义代理的请求头，具体用法我也不是很清楚，照着复制
                   proxy_redirect      off;
                   proxy_set_header    Host    $host;
                   proxy_set_header    X-Real-IP       $remote_addr;
                   proxy_set_header REMOTE-HOST $remote_addr;
                   proxy_set_header    X-Forwarded_For $proxy_add_x_forwarded_for;
                   proxy_connect_timeout       90; #连接超时
                   proxy_send_timeout          90; #请求超时
                   proxy_read_timeout          90; #读取超时
                   proxy_buffer_size           4k;
                   proxy_buffers               4 32k;
                   proxy_busy_buffers_size     64k;
                   proxy_temp_file_write_size  64k;
                   client_max_body_size        500m; #客户端最大延迟
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
   # 多个server
   #    server {
   #       listen       9300;
   #        server_name  localhost;
   #
   #        location / {
   #	    root   /dist; 
   #            index  index.html index.htm;
   #       }
   		
   #	location /api/ {
   #		proxy_set_header Host $host;
   #		proxy_set_header X-Forwarded-For $remote_addr;
   #		proxy_pass http://127.0.0.1:8484/api/;
   #		proxy_send_timeout 300;
   #		proxy_read_timeout 300;
   #		proxy_connect_timeout 300;
   #	}
      # location ^~/api { 
            #   rewrite ^/api/(.*)$ /$1 break; 
             #  proxy_pass http://localhost:8081; 
       # }
   
    #       error_page   500 502 503 504  /50x.html;
    #       location = /50x.html {
    #          root   html;
    #      }
    #   }
       
       # another virtual host using mix of IP-, name-, and port-based configuration
       #
       #server {
       #    listen       8000;
       #    listen       somename:8080;
       #    server_name  somename  alias  another.alias;
   
       #    location / {
       #        root   html;
       #        index  index.html index.htm;
       #    }
       #}
   
   
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

   2.vue存放路径和上面root路径一样

   ![](C:\Users\Admin\Desktop\2021工作文件\img\vue.PNG)

   3.java项目路径存在代理的tomcat下的webapps下

   ![捕获](C:\Users\Admin\Desktop\捕获.PNG)

#### 5.nginx平滑升级

##### 	1.安装新版本nginx

```
cd /usr/local/source
wget  http://nginx.org/download/nginx-1.9.0.tar.gz
tar -zxvf nginx-1.6.2.tar.gz
mv nginx-1.6.2 /usr/local/nginx
cd /usr/local/nginx/
```

#####    2.预编译nginx

```
cd /usr/local/nginx/nginx-1.xx.x
./configure --user=www --group=www --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module
```

#####    3.make编译(注意不要make install)

```
make 
```

   4.对nginx文件进行更新

```
cd /usr/local/nginx/sbin
# 进入nginx的执行文件目录内
mv nginx nginx.old
# 将旧版本Nignx执行文件备份为nginx.old
cp /root/nginx-1.16.0/objs/nginx .
# 将新版本Nginx执行文件移动到当前目录内
```

#####   4.进行平滑重启	

```
/usr/local/nginx/sbin/nginx -t
# 检测新版本Nginx是否正常 正常为 successful
ps -ef|grep nginx
# 查看旧版本nginx进程
```

![img](https://pics2.baidu.com/feed/2934349b033b5bb5c1398cc4f8b2323fb700bc82.jpeg?token=ac28f7e2c63243bb6dda0c87b5787759)

```
kill -USR2 4428
# 向主进程发送USR2信号，Nginx会启动一个新版本的master进程和工作进程，和旧版一起处理请求
```

![img](https://pics2.baidu.com/feed/d058ccbf6c81800aef45bd127c54d4fc838b47f6.jpeg?token=11b1ade63911213ff5238fc9fef54777)

```
此时再次查看Nginx进程就发现有俩Nginx在工作

kill -WITCH 4428
# 向原Nginx主进程发送WINCH信号，它会逐步关闭旗下的工作进程（主进程不退出），这时所有请求都会由新版Nginx处理
kill -QUIT `cat /usr/local/nginx/logs/nginx.pid.oldbin`
# 杀死旧版本Nginx主进程或者 kill -9 2248 也可以
/usr/local/nginx/sbin/nginx -v
# ouput：nginx version: nginx/1.20.2
```

