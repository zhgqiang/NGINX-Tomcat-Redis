NGINX-Tomcat-Redis-
===================

实现NGINX+Tomcat+Redis一起进行集群，进行分布处理用户的信息

NGINX配置文件思路就是当客户端访问tomcat（因为其就是一个服务器）的时候， 
就会产生一个sessionid，可以作为key存到redis，value可以是个user，集合等，
我用的是用户，因为负载均衡，当我再次发送请求的时候不一定访问的是当前的tomcat，
可能是另一个tomcat，因为在访问不同的服务器的时候，会产生不同的sessionid，
其存是需要解决的，所以我就将在第一次访问的时候就产生的sessionid存到cookie里，
再次访问的时候因为cookie是存在客户端的，取出cookie里的sessionid，
用此sessionid取出redis里第一次访问存的用户名与这次访问的用户名做比较，如果相同直接进行连接。

user root; #用户名
worker_processes  3;
pid logs/nginx.pid;
# [ debug | info | notice | warn | error | crit ]
error_log   /usr/local/nginx/logs/error.log  info;#与自己文件目录一样 
events {
worker_connections   1024;
# use [ kqueue | rtsig | epoll | /dev/poll | select | poll ] ;
# 具体内容查看 http://wiki.codemongers.com/事件模型
use epoll;
}

http {
include       mime.types;
default_type  application/octet-stream;

log_format main      '$remote_addr - $remote_user [$time_local]  '
'"$request" $status $bytes_sent '
'"$http_referer" "$http_user_agent" '
'"$gzip_ratio"';

log_format download  '$remote_addr - $remote_user [$time_local]  '
'"$request" $status $bytes_sent '
'"$http_referer" "$http_user_agent" '
'"$http_range" "$sent_http_content_range"';

client_header_timeout  3m;
client_body_timeout    3m;
send_timeout           3m;

client_header_buffer_size    1k;
large_client_header_buffers  4 4k;

gzip on;
gzip_min_length  1100;
gzip_buffers     4 8k;
gzip_types       text/plain;

output_buffers   1 32k;
postpone_output  1460;

sendfile         on;
tcp_nopush       on;
tcp_nodelay      on;
#send_lowat       12000;

keepalive_timeout  75 20;

#lingering_time     30;
#lingering_timeout  10;
#reset_timedout_connection  on;

 upstream tomcats { 
             server 192.168.214.130:8080 weight=1; 
            server 192.168.214.130:8080 weight=1；
            # ip_hash;
            }
server {
        listen        9999; #nginx监听的端口
        server_name   localhost; #nginx安装的虚拟机地址
        access_log  /usr/local/nginx/logs/access.log  main;


location / {
        proxy_pass         http://tomcats; #与上边upstream “tomcats”一致
        proxy_redirect     off;

        proxy_set_header   Host             $host;
        proxy_set_header   X-Real-IP        $remote_addr;
        #proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
        client_max_body_size       10m;
        client_body_buffer_size    128k;

        client_body_temp_path      /usr/local/nginx/client_body_temp;

        proxy_connect_timeout      90;
        proxy_send_timeout         90;
        proxy_read_timeout         90;
#       proxy_send_lowat           12000;

        proxy_buffer_size          4k;
        proxy_buffers              4 32k;
        proxy_busy_buffers_size    64k;
        proxy_temp_file_write_size 64k;

#       proxy_temp_path            /usr/local/nginx/proxy_temp;

        charset  gb2312;
}

#error_page  404  /404.html;
#location /404.html {
#root  /spool/www;
#charset         on;
#source_charset  koi8-r;
#}

location /old_stuff/ {
        rewrite   ^/old_stuff/(.*)$  /new_stuff/$1  permanent;
}

location /download/ {

valid_referers  none  blocked  server_names  *.example.com;

if ($invalid_referer) {
        #rewrite   ^/   http://www.example.com/;
        return   403;
}

#rewrite_log  on;
# rewrite /download/*/mp3/*.any_ext to /download/*/mp3/*.mp3
rewrite ^/(download/.*)/mp3/(.*)\..*$
/$1/mp3/$2.mp3                   break;

root   /usr/local;

#autoindex    on;
#access_log   /var/log/nginx-download.access_log  download;
}

location ~* ^.+\.(jpg|jpeg|gif)$ {
root   /usr/local;
        access_log   off;
        expires      30d;
        }
 }
 }
redisdemo工程配置及代码
web.xml配置：
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://java.sun.com/xml/ns/javaee" xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd" id="WebApp_ID" version="3.0">
  <servlet>
  	<servlet-name>RedisDemo</servlet-name>
 	<servlet-class>com.tarena.redis.RedisServlet</servlet-class>
  </servlet>
  <servlet-mapping>
  	<servlet-name>RedisDemo</servlet-name>
  	<url-pattern>*.do</url-pattern>
  </servlet-mapping>
</web-app>

测试程序：
package com.tarena.redis;

import java.io.IOException;
import java.io.PrintWriter;

import javax.servlet.ServletException;
import javax.servlet.http.Cookie;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;

import redis.clients.jedis.Jedis;

public class RedisServlet extends HttpServlet {
	@Override
	protected void service(HttpServletRequest req, HttpServletResponse res)
			throws ServletException, IOException {
		res.setContentType("text/html;charset=utf-8");
		PrintWriter out = res.getWriter();
		String uri = req.getRequestURI();
		String action = uri.substring(uri.lastIndexOf("/"),
				uri.lastIndexOf("."));
		Jedis redis = new Jedis("192.168.214.130", 6379);
		String serverName = req.getServerName();
		if ("/login".equals(action)) {
			HttpSession session = req.getSession();
			String sessionId = session.getId();
			Cookie c = new Cookie("cookieName", sessionId);
			redis.set(sessionId, req.getParameter("userName"));
			res.addCookie(c);
			out.println("nginx:" + serverName + " | "
					+ "tomcat:192.168.214.130  添加redis key"
					+ req.getParameter("userName"));
		} else if ("/main".equals(action)) {
			Cookie[] cookies = req.getCookies();
			if (cookies != null) {
				for (Cookie c : cookies) {
					if (c.getName().equals("cookieName")) {
						String sessionId = c.getValue();
						if (req.getParameter("userName").equals(
								redis.get(sessionId))) {
							out.println("nginx:" + serverName + " | "
									+ "tomcat:192.168.214.130  取得redis key"
									+ redis.get(sessionId));
						} else {
							out.println("nginx:" + serverName + " | "
									+ "no key");
						}
					}
				}
			} else {
				req.getRequestDispatcher("login.jsp").forward(req, res);
			}
		}
		out.close();
	}
}

RESTClient测试
http://192.168.214.130:9999/redisdemo/login.do?userName=abcd进入

输入value 点击登陆，将value存入了redis 页面显示：
nginx:192.168.214.130 | tomcat:192.168.214.130 添加redis keyabcd
http://192.168.214.130:9999/redisdemo/main.do?userName=abcd进入，实现负载均衡，页面分别显示
nginx:192.168.214.130 | tomcat:192.168.214.130 取得redis keyabcd
nginx:192.168.214.130 | tomcat:192.168.214.132 取得redis keyabcd
