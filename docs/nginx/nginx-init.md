# nginx

## nginx 基本概念

*Nginx* (engine x) 是一个高性能的HTTP和反向代理web服务器。

Nginx 的最重要的几个使用场景：

1. 静态资源服务，通过本地文件系统提供服务；（搭建静态页面，比如图片服务、博客服务、官网展示服务）
2. 反向代理服务，延伸出包括缓存、负载均衡等；
3. tcp 代理等....



## nginx 服务搭建（基于ubuntu）

```sh
apt-get install nginx
```

访问 `curl localhost` 得到 nginx 的html 欢迎页信息即部署成功。


## nginx 常用命令

```sh
# 查看 nginx 版本号
nginx -v 
# 启动 nginx
nginx 
# 启动 nginx 并指定配置文件
nginx -c /opt/nginx/nginx-1.20.1/conf/nginx.conf
# 快速停止或关闭Nginx
nginx -s stop
# 正常停止或关闭Nginx
nginx -s quit
# 校验配置是否正确
nginx  -t
# 重新加载配置文件
nginx -s reload 

```








