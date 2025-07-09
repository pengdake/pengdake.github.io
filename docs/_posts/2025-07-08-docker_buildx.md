---
layout: single
toc: true
toc_sticky: true
categories: [docker]
tags: [docker, image, docker_swarm]
---


# 使用docker_buildx构建跨平台容器镜像

## 背景
最近用旧的笔记本和树莓派4b搭建了一个两节点的docker swarm环境，由于笔记本和树莓派的cpu架构不同，分别是amd64和arm的，而为了实现部署应用的容器能够在所有节点自由调度，需要保证节点获取与其处理器架构对应的容器镜像。因此需要利用docker buildx同时制作跨平台的容器镜像

## 安装
qemu模拟器安装启用，用于非本机架构的镜像构建
```
docker run --rm --privileged tonistiigi/binfmt --install all
```
```
latest: Pulling from tonistiigi/binfmt
07872a4bb80c: Pull complete
da5179bfe27f: Pull complete
Digest: sha256:1b804311fe87047a4c96d38b4b3ef6f62fca8cd125265917a9e3dc3c996c39e6
Status: Downloaded newer image for tonistiigi/binfmt:latest
installing: mips64le OK
installing: amd64 OK
installing: 386 OK
installing: s390x OK
installing: ppc64le OK
installing: riscv64 OK
installing: mips64 OK
installing: loong64 OK
{
  "supported": [
    "linux/arm64",
    "linux/amd64",
    "linux/amd64/v2",
    "linux/riscv64",
    "linux/ppc64le",
    "linux/s390x",
    "linux/386",
    "linux/mips64le",
    "linux/mips64",
    "linux/loong64",
    "linux/arm/v7",
    "linux/arm/v6"
  ],
  "emulators": [
    "qemu-i386",
    "qemu-loongarch64",
    "qemu-mips64",
    "qemu-mips64el",
    "qemu-ppc64le",
    "qemu-riscv64",
    "qemu-s390x",
    "qemu-x86_64"
  ]
}

```

docker buildx在版本19.03后的docker已经内置了，只是需要执行以下命令创建构建器的容器
创建buildx
```
docker buildx create --use --name multiarch 
```
```
#注：如果上传的本地仓库为http,则需要按以下步骤初始化
cat <<EOF > buildkitd.toml
[registry."192.168.31.98:5000"]
  http = true
  insecure = true
EOF
docker buildx create \
  --name multiarch \
  --config ./buildkitd.toml \
  --use
```
运行buildx
```
docker buildx inspect --bootstrap
```
查看
```
docker buildx inspect multiarch 
```
```
Name:          multiarch
Driver:        docker-container
Last Activity: 2025-07-08 04:24:18 +0000 UTC

Nodes:
Name:                  multiarch0
Endpoint:              unix:///var/run/docker.sock
Status:                running
BuildKit daemon flags: --allow-insecure-entitlement=network.host
BuildKit version:      v0.22.0
Platforms:             linux/arm64, linux/arm/v7, linux/arm/v6
Labels:
 org.mobyproject.buildkit.worker.executor:         oci
 org.mobyproject.buildkit.worker.hostname:         9929657bf10d
 org.mobyproject.buildkit.worker.network:          host
 org.mobyproject.buildkit.worker.oci.process-mode: sandbox
 org.mobyproject.buildkit.worker.selinux.enabled:  false
 org.mobyproject.buildkit.worker.snapshotter:      overlayfs
GC Policy rule#0:
 All:           false
 Filters:       type==source.local,type==exec.cachemount,type==source.git.checkout
 Keep Duration: 48h0m0s
GC Policy rule#1:
 All:           false
 Keep Duration: 1440h0m0s
 Keep Bytes:    9.313GiB
GC Policy rule#2:
 All:        false
 Keep Bytes: 9.313GiB
GC Policy rule#3:
 All:        true
 Keep Bytes: 9.313GiB

```

## 制作镜像示例
```
# 制作amd64和arm64架构的镜像，并上传到本地仓库，注意要指定builder为上一步创建的multiarch
docker buildx build --builder multiarch --platform linux/amd64,linux/arm64 -t 192.168.31.98:5000/go-video --push .
```

## 扩展
如果是已有现成的异构镜像，可通过创建manifest的方式来实现
假如本地仓库存在以下两个异构镜像
```
# arm64
192.168.31.98:5000/nginx:1.29.0-arm64
# amd64
192.168.31.98:5000/nginx:1.29.0-amd64
```
创建对应的manifest，并上传到本地仓库
```
docker manifest create --insecure   192.168.31.98:5000/nginx:1.29.0 --amend 192.168.31.98:5000/nginx:1.29.0-arm64 --amend  192.168.31.98:5000/nginx:1.29.0-amd64
docker manifest push --insecure 192.168.31.98:5000/nginx:1.29.0
```
验证镜像的manifest已包含两种架构
```
docker manifest inspect --insecure 192.168.31.98:5000/nginx:1.29.0
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.list.v2+json",
   "manifests": [
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 1778,
         "digest": "sha256:ccde53834eab53e85b35526a647cdb714ea4521b1ddf5a07b5c8787298d13087",
         "platform": {
            "architecture": "amd64",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 1778,
         "digest": "sha256:a27d6dc7e214fa0a360a2e3dbe1aa36b7ab00b76915df7a1d5feb6556856f700",
         "platform": {
            "architecture": "arm64",
            "os": "linux",
            "variant": "v8"
         }
      }
   ]
}

```