# arthas

## arthas 下载按照

下载 `arthas-boot.jar`，然后用java -jar的方式启动：

```shell
curl -O https://arthas.aliyun.com/arthas-boot.jar
java -jar arthas-boot.jar
```


**http 访问 500**
```shell
watch com.example.arthas.TestController * '{params, throwExp}' -e
```

**http 访问 404**
```shell
trace javax.servlet.Servlet * > servlet.txt
```

**http 访问 401**
```shell
trace javax.servlet.Filter 
```