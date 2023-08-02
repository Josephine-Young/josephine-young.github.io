# Nginx

#### Introduction:

占用内存少，并发能力强

支持HTTP/IMAP/POP3/SMTP服务

使用ip_hash策略进行负载均衡

多个客户端 --> 代理服务器 --> 多个服务器

**正向代理**：代理客户端，e.g. VPN

**反向代理**：在有多台服务器分布时，代理服务器端

#### 配置文件详解
文件名：nginx.conf
##### 组成结构

```shell
# 全局块，可以配置运行nginx服务器的用户组，进程pid运行文件与日志存放路径，配置文件引入，允许生成worker process（进程）数等
# user administrator;
# worker_processes 2;
# pid /nginx/pid/nginx.pid;
# error_log logs/error.log

events{
# 配置影响nginx服务器或与用户的网络连接
# accept_mutex on; # 设置网路连接序列化，防止惊群现象发生，默认为on
# multi_accept on; # 设置一个进程是否同时接受多个网络连接，默认为off
# use epoll; # 事件驱动模型 select|poll|kqueue|epoll|resig|/dev/poll|eventport
worker_connections 1024; #最大连接数
}
http{
# 可以配置多个server，代理、缓存、日志定义等
include mime.types #文件扩展名与文件类型映射表
default_type application/octet-stream #默认文件类型
# access_log off
access_log log/access.log main;
send_file 	on;
send_file_max_chunk 100k;
keepalive_timeout 65; #超时时间
error_page 404 https://www.baidu.com; #错误页
	server{
	# 配置虚拟主机的参数
	keep_alive_requests 120; # 单连接请求次数上限
	listen 4545;
	server_name 127.0.0.1;
		location{
		# 配置请求的路由以及各种页面的处理情况
		}
	}
}
```
**惊群现象**： 一个网路连接到来时，多个睡眠的进程被同时叫醒，但只有其中一个进程能获得链接，会影响系统性能；

##### 反向代理的设置

```shell
http{
	server{
		listen		80; #监听80端口，将所有被反向代理的服务器都指向该端口
		server_name localhost;
		location / { #请求的url过滤，采用正则匹配；
					 #匹配/目录，客户端在访问该目录时经过nginx服务器
			root	html; # 根目录
			index	index.html index.htm; # 默认页
			proxypass	http://127.0.0.1:8080;
			proxypass 	http://127.0.0.1:8081; #反向代理本机8080与8081端口
			# deny all
			deny 127.0.0.1 # 拒绝访问的ip
			allow 172.18.5.54 # 允许访问的ip
		}
	}
}
```
##### 使用upstream配置反向代理的服务器组

```shell
http{
	upstream myserver{
		server 192.168.0.100
		server 192.168.0.127
	}
	server {
		listen 80;
		server_name 127.0.0.1;
		
		location / {
			root html;
			index index.html;
			proxy_pass http://myserver; #将请求转向myserver对应的服务器列表
		}
	}
}
```

#### 负载均衡
内置策略：热备(backup)、轮询(default)、加权轮询、IP哈希、url哈希等方法
##### IP哈希
对客户端ip进行hash操作，根据hash结果将同一个客户端ip发出的请求分发给同一台服务器处理

#### 参考链接

[]: https://zhuanlan.zhihu.com/p/35705029	" Nginx最全使用教程"

[]: https://www.runoob.com/w3cnote/nginx-setup-intro.html "Nginx配置详解"
