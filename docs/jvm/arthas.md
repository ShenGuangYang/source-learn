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





## arthas 在eclipse-temurin进行下调试



```shell
apk update && apk add bash curl busybox-extras jattach && wget -qO /tmp/arthas.zip "https://maven.aliyun.com/repository/public/com/taobao/arthas/arthas-packaging/3.7.1/arthas-packaging-3.7.1-bin.zip" && mkdir -p /opt/arthas && unzip /tmp/arthas.zip -d /opt/arthas && jattach 1 load instrument false /opt/arthas/arthas-agent.jar && java -jar /opt/arthas/arthas-client.jar 127.0.0.1 3658
```