---
title: Nginx分流(一条请求复制成多条请求)
tags: 
- Nginx
---



# 需求

```less
由A服务器发送一条请求(POST)，复制成两条请求以不同的URL同时转发给B服务器和C服务器。
例子：A发送 http://a.com/a 经过Nginx分别向B发送 http://b.com/b 和C发送 http://c.com/c

```

# 谈下试错过程

```markdown
1. 使用反向代理，无法复制请求
2. 使用重定向，post内容不支持，同时无法复制请求
3. 使用upstream来复制访问请求，同时给自己多个不同端口来拦截，拦截后反向代理，并没有被拦截到，这种方式属于负载均衡一类的，无法做到同时发送
```

# 最后方案

```markdown
使用nginx提供的 mirror模块，其实这种需求有个专业的叫法：**引流测试**。你搜下这个就会有很多相关的文章了。
```

# 配置实现

```bash
最终配置也很简单，在正常的反向代理部分写上 mirror /mirror  #复制子请求的拦截部分，再在下面定义另个location /mirror 即可。
```

# 配置文件

```ini
include       mime.types;
default_type  application/octet-stream;
sendfile        on;
keepalive_timeout  65;
upstream self{
# 如果需要使用ip的可以反向代理时候使用这部分内容
        server 192.168.0.74:8080;
}

server {
    listen       8888;
    server_name  a.com;
	location /a {
                mirror /mirror;
                # 需要放大流量 再加一个 mirror /mirror;
                proxy_pass http://b.com/b;
	}
	
	location = /mirror {
		internal;
		proxy_pass http://c.com/c;
	}
}
```

# 扩展

- Nginx官方mirror介绍：https://nginx.org/en/docs/http/ngx_http_mirror_module.html
- 性能工具之常见流量复制工具：[https://segmentfault.com/a/1190000039982660](https://link.juejin.cn/?target=https%3A%2F%2Fsegmentfault.com%2Fa%2F1190000039982660)
- Nginx的upstream详解：[www.jianshu.com/p/8671c40a5…](https://link.juejin.cn/?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2F8671c40a5be8)
- Nginx在线配置生成工具：[www.nginxedit.cn/](https://link.juejin.cn/?target=https%3A%2F%2Fwww.nginxedit.cn%2F)