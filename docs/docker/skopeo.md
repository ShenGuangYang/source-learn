# skopeo

## skopeo 简介

主要是针对镜像仓库、容器镜像之间的各种操作。

本文介绍镜像两个镜像仓库的同步操作

## 两个镜像仓库同步

### 命令同步

将dockerhub的镜像同步至阿里云仓库

```shell
skopeo sync --src docker --dest docker docker.io/apache/iotdb:1.3.0-standalone registry.cn-shanghai.aliyuncs.com/xxx-name --all
```

### 配置文件同步

创建配置文集

```yaml
docker.io:
  images-by-tag-regex:
    apache/iotdb: ^1\.[01]\.[012]-standalone$
  tls-verify: false
```

执行同步命令

```shell
skopeo sync --src yaml --dest docker sync.yml registry.cn-shanghai.aliyuncs.com/jsmaxtropy --all
```

## 参考链接

[GitHub - containers/skopeo](https://github.com/containers/skopeo)

[最佳镜像搬运工 Skopeo 指南（1）-阿里云开发者社区](https://developer.aliyun.com/article/1113538?spm=a2c6h.24874632.expert-profile.469.26106bd00jDcbd#slide-20)

[最佳镜像搬运工 Skopeo 指南（2）-阿里云开发者社区](https://developer.aliyun.com/article/1113540?spm=a2c6h.24874632.expert-profile.468.26106bd00jDcbd#slide-15)
