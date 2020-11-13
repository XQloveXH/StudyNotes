## 一、前言
Nginx是什么？
Nginx是一个开源的Web服务器，同时Nginx也提供了反向代理和负载均衡的功能。
Nginx通常作为负载均衡器暴露在外网接受用户请求，同时也使用其反向代理的功能，将用户的请求转发到实际提供服务的内网服务器。

Windows什么情况下需要Nginx？
通常来说Windows下IIS就够用了，支持 .NET、ASP、PHP等等s，不过如果你需要做负载均衡那你就需要Nginx，或者说你在一台服务器上，部署了Apache、IIS、Tomcat等多个Web服务器，这时候把80端口或443端口给Nginx在合适不过了。

当然，作为商业公司来说，通常不会有以上情况，但是如果你是个草根站长。或者说你想把你的Windows开发机作为服务器对外提供服务，那把你的Windows装上Nginx再合适不过了。

本篇环境信息
工具/环境	版本
Windows	Windows 10 - 1803
Nginx	1.15.2

![07-nginx概念回顾](G:\soft\Typora\img\07-nginx概念回顾.png)



## 二、安装过程
下载Nginx
官方下载地址：http://nginx.org/en/download.html

这次我们下载1.15.2版本：http://nginx.org/download/nginx-1.15.2.zip

下载后，将nginx-1.15.2.zip解压到 C:\Tools\Nginx

请注意，Nginx目录所在的路径中不要有中文字符，也不建议有空格。

启动Nginx
使用CMD命令start命令启动nginx

c: && cd c:\tools\nginx
start nginx

如果开启了Windows防火墙，记得允许访问网络。



启动成功后，浏览器访问 localhost，即可看到Nginx 欢迎页



如果启动启动失败，可能是IIS占用了80端口。去掉IIS监听的80端口即可。

##  Nginx常用配置
配置文件说明
Nginx所有配置文件都在Nginx根目录conf子目录中（C:\Tools\Nginx\conf）
Nginx核心配置文件： C:\Tools\Nginx\conf\nginx.conf

我们的常用配置只需要在nginx.conf中调整server节点就可以了
在nginx.conf文件末尾有如下示例



```nginx


worker_processes  1;

events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;



    server {
        listen       81;
        server_name  localhost;

        
    }
	
	server{
		
		listen		9001;
		server_name localhost;
		
		location ~ /eduservice/ {
			proxy_pass http://localhost:8001;
		}
		location ~ /oss/ {
			proxy_pass http://localhost:8002;
		}
		
	
	} 

}

```

这个示例我们保留即可，我们配置反向代理、负载均衡直接在后面追加即可

![08-nginx配置请求转发](G:\soft\Typora\img\08-nginx配置请求转发-1594724199600.png)

##注意事项

在windows下安装的nginx启动后，关闭cmd窗口nginx仍然在后台运行。

需要使用**nginx -s stop**关闭

### Nginx常用命令说明
命令	说明
nginx -h	查看帮助信息
nginx -v	查看Nginx版本
nginx -s stop	停用Nginx
nginx -s quit	优雅的停用Nginx（处理完正在进行中请求后停用）
nginx -s reload	重新加载配置，并优雅的重启进程
nginx -s reopen	重启日志文件
## 备注
本篇参考
https://nginx.org/en/docs/windows.html

延伸阅读
将Nginx作为Windows服务并开机启动：http://blog.haoji.me/windows-nginx-server.html
