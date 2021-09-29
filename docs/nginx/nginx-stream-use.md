# nginx 使用 stream 做转发

## 配置文件加一下配置

配置说明如下，监听 8080 端口，转发到 192.168.0.100:8080

```
stream {
    upstream tcp-demo {
        server 192.168.0.100:8080;
    }
   server {
      listen 8080;
      proxy_connect_timeout 1s;
      proxy_pass tcp-demo;
   }
}
```



检查配置文件是否正确并重新加载配置文件

```sh
# 检查配置文件是否正确
nginx -t
# 重新加载配置文件
nginx -s reload
```

