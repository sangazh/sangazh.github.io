---
layout: post
title: 记一次不知道为何解决了的docker网络问题
tags:
  - Docker
  - network
  - linux
excerpt_separator: <!--more-->
---

公司断了几次电后，发现我们开发服务器的docker pull不起作用了。

提示错误：

```
docker: Error response from daemon: Get https://registry-1.docker.io/v2/: dial tcp: lookup registry-1.docker.io
```

搜索后大概有两个解决方法。
1. 加dns
2. 加代理

先说第一个：
`/etc/resolv.conf`添加两行，没有就新建。

```
nameserver 8.8.8.8
nameserver 8.8.4.4
```

第二个是在systemd运行docker时，把配置的代理的env加载上去。
<!--more-->
新建文件：`/etc/systemd/system/docker.service.d/10_docker_proxy.conf`， 内容为：
```
[Service]
Environment=HTTP_PROXY=http://1.1.1.1:111
Environment=HTTPS_PROXY=http://1.1.1.1:111
```
>这里的`1.1.1.1:111`是例子，要按实际修改。

接下来重启docker

```
sudo systemctl daemon-reload
sudo systemctl restart docker
```

>注意两种配置修改后都要重启docker哈。

由于第二种我不知道代理服务器应该配什么，我只配了dns就好了，可以正常pull了...

...

然而第二个错误提示出现了：

```
docker: error pulling image configuration: Get https://production.cloudflare.docker.com/registry-v2/docker/registry/v2/blobs/sha256/c2/xxxxxx dial tcp `some ip`: connect: connection timed out.
```

中英文搜了搜，感觉可能真的需要配个国内镜像，比如`DaoCloud`的
曝光率蛮高的。于是我决定试试，挺方便，照着官网**Linux**那栏下个脚本跑跑就能自动修改`/etc/docker/daemon.json`并自动重新docker了。

再来看看修改后的 `/etc/docker/daemon.json` 文件：

```
{
  "registry-mirrors": ["http://f1361db2.m.daocloud.io"],
  "debug": true,
  "experimental" : true,
  "insecure-registries" : [
    "192.168.1.101:5000"
  ]
}
```

ok，再来 `docker pull`一下。提示说，要拉的image版本过低。所以这镜像我用不了呗。没办法我只好把 `registry-mirrors`这行配置删掉，手动再重启一遍docker。

```
sudo systemctl daemon-reload
sudo systemctl restart docker
```

理论上说，我应该回到了错误2的节点。但是，报错好了！

对，不知道怎样，总之网络问题解决了。

实际我只做了添加dns，和重启了三遍docker，而已。


#### References
- [加dns的forum](https://forums.docker.com/t/error-response-from-daemon-get-https-registry-1-docker-io-v2/23741/18)
- [加代理的stackoverflow](https://stackoverflow.com/questions/46036152/lookup-registry-1-docker-io-no-such-host/46037636)
- [DaoCloud](https://www.daocloud.io/mirror)
- [docker 配置国内镜像源 linux/mac/windows](https://www.jianshu.com/p/9fce6e583669)