+++
title = "Docker Registry GC"
date = 2018-06-18T00:31:16+08:00
draft = false

tags = ["docker"]
categories = []

summary = "garbagecollect 删除 untagged 镜像"

[header]
image = "https://terminus-jferic.oss-cn-shanghai.aliyuncs.com/68747470733a2f2f7777772e646f636b65722e636f6d2f73697465732f64656661756c742f66696c65732f6f79737465722d72656769737472792d332e706e67.png"
caption = ""
preview = true

+++

我们的 CI 平台中私有化部署了 Docker Registry 用来存储 CI 过程中的镜像。这些镜像主要包括用于加速的依赖包缓存镜像和用于部署的应用镜像。但是官方提供的 Docker Registry 没有自动清理镜像的功能，导致 Registry 中的镜像会随着时间越积越多，对 Registry 的 Storage 造成了压力。

考虑到 CI 平台的特性，随着应用不断打包部署，缓存镜像和应用镜像都在不断更新，历史版本的镜像是可以删除的。所以，我们尝试进行 Docker Registry GC。

## GC 的方法及问题
官方文档提供的 Delete 接口是基于镜像 Manifest 的 Digest 去删除镜像。

```http
DELETE /v2/<name>/manifests/<reference>
```

> For deletes, `reference` *must* be a digest or the delete will fail.

我们首先需要获取到所有待删除镜像的 Digest。

```go
// 首先遍历 Repositories 先获取 Tags
tags, err := registry.GetTotalRepositoryTags(env.DockerRegistry, *repo, *outputFile, env.TagsIncludes...)
... // handler error

for _, tag := range tags {
	// 然后获取该 Tag 的 Digest, by 'Docker-Content-Digest' Header
	digest, err := registry.GetTagDigest(env.DockerRegistry, *repo, tag)
	... // handler error

	// 最后使用 Digest 删除镜像
	err = registry.DeleteDigest(env.DockerRegistry, *repo, tag, digest)
	... // handler error
}
```

最后使用 Registry 的 garbage-collect 命令清理镜像。

```bash
bin/registry garbage-collect [--dry-run] /path/to/config.yml
```

下图是 dry-run 的结果，仅仅 57 个 blobs 被标志成可以删除，和预期的相差甚远。

![image-20180618162303741](https://terminus-jferic.oss-cn-shanghai.aliyuncs.com/image-20180618162303741.png)



### Overwritten tags 导致 blobs 未被 GC  

通过查找缓存镜像 c23b08159e8e2979eeab3c3b43c46132/release 发现该 repositry 下竟然存在 manifest ，并标记了一堆相关的 blobs。

![image-20180618164316360](https://terminus-jferic.oss-cn-shanghai.aliyuncs.com/image-20180618164316360.png)

查看 Registry 的 tag 目录，却并不存在任何 tag。

![image-20180618164348321](https://terminus-jferic.oss-cn-shanghai.aliyuncs.com/image-20180618164348321.png)

通过 ha256:1bc8a50d8403ed7464cd97bfd41eb6d322f611e89462600068929249210f0d84 这个 digest 真实能够获取到 manifest 文件。

![image-20180618164814375](https://terminus-jferic.oss-cn-shanghai.aliyuncs.com/image-20180618164814375.png)

这样基本确定：使用 tag 获取 digest 进而删除镜像的方法遗漏了一些 Manifests。

#### Untagged manifests

结合我们的使用场景，在官方找到很多相关的 Issues，例如：

[[Issue] Registry garbage-collect does not clean up blobs from overwritten tags](https://github.com/docker/distribution/issues/2212)

[[Issue] Feature req: garbage-collector should have possibility to clean untagged revisions](https://github.com/docker/distribution/issues/2301)

确定原因是：

> 我们的缓存镜像使用 latest 作为 tag，每次打包都可能生成一个新的镜像 push 到 Registry 中，这样就 overwritten tags，在 Repository 的 revisions 目录下仍然保存了历史的 digests。
>
> 即使我们删除 latest tag 标记的镜像，但因为历史的 untagged manifest 仍然关联着 blobs，所以大量的 blobs 并没有被删除。

使用社区的 patch 版本（Registry/2.7 将  release 该功能）

[[PR] add possibility to clean untagged manifests](https://github.com/docker/distribution/pull/2302)

可以删除 untagged 的 manifest 及其关联的 blobs。

![image-20180618162417130](https://terminus-jferic.oss-cn-shanghai.aliyuncs.com/image-20180618162303741.png)

下面我们简单了解一下 garbage collect 的挑战。

## Registry GC 的挑战

Registry 的物理存储包括两部分：blobs 和 repositories。其中 blobs 存储了镜像的 layers 和 manifest 数据文件，用 digest 作为 index；repositories 目录存储了个各种 Meta 数据，包括 layers、manifest 的 digest 信息。

![image-20180618202642208](https://terminus-jferic.oss-cn-shanghai.aliyuncs.com/image-20180618202642208.png)

基本关系是 repositry -> manifest (tag)  -> layer => blob

一个 repository 下面包含一个或者多个 manifest，可读的 tag 与manifest 关联； 一个 manifest 文件包含一个或者多个 layer；manifest 和 layer 的数据文件都存储在 blobs 目录下，以 digest 作为子目录索引 blob 数据文件。

镜像的特点是分层 （layers）存储，镜像间可以共享 layer。这样删除镜像的核心挑战变成如何在 多个镜像（manifest）共享数据（layer）的情况下保证数据一致，同时保证并发写操作的原子性。

#### Garbage collect

v2.6 版本的 Docker Registry 当前实现：

> Just find all the manifests, enumerate the referenced blobs and delete the blobs not in that set.

所以实现了 mark 和 sweep 两个阶段，本质上是标记&清理的 Lock the World GC 机制。因为在 mark 之后如果还有新的写入（没有mark）将导致误删。

### Deletes on Roadmap

[Delete section on Roadmap](https://github.com/docker/distribution/blob/master/ROADMAP.md#deletes) 重点讨论了删除背后的问题，也提出了 "Reference Counting", "Generational GC", Centralized Oracle" 等方案。本质是 **引用计数**  问题，同时需要在删除时保证数据一致性。