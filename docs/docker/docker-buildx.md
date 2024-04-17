# docker 编译跨平台镜像

## 安装qemu

在特定架构系统上构建不同的架构平台的镜像是使用的qemu虚拟技术. 所以,需要安装与之相关的这两个工具

```shell
sudo apt install -y qemu-user-static binfmt-support
```

安装好qemu后，查看支持的架构平台

```shell
docker buildx ls
```

## 构建多架构镜像

```shell
docker buildx create --use --name mybuilder
docker buildx build --builder=mybuilder --platform=linux/arm64,linux/amd64 -t xxx/tag -f ./Dockerfile . --push
docker buildx rm mybuilder
```

## 问题处理

如果你用`docker`驱动来构建多平台镜像,会得到以下错误，因为buildx默认使用的是`docker`驱动,这个是不支持多平台构建的.所以我们需要切换使用`--builder=mybuilder`驱动

```
ERROR: multiple platforms feature is currently not supported for docker driver. Please switch to a different driver (eg. "docker buildx create --use")
```
