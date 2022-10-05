---
layout: post
title: 使用SSH连接Github失败
subtitle: 
categories: 工具
tags: []
---

# 起因
之前在某个项目中使用Git将本地代码push到了Github，一直没出现任何问题，在隔了一段时间后再次使用Git push代码，却出现了以下图片出现的错误。![](https://blogimg-1253107768.cos.ap-guangzhou.myqcloud.com/blogImage/ssh01.png#crop=0&crop=0&crop=1&crop=1&id=SLrGp&originHeight=142&originWidth=820&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
在尝试使用  `ssh -v git@github.com` 后发现果然连接不上，同样提示 Permission denied。在进入 `~/.ssh/` 文件夹后发现相关的私钥也存在，这就很奇怪了。
# 解决方法
首先说一下我的错误原因，在 ~/.ssh/ 文件夹下有个 config 文件，这个文件中配置的是你的密钥信息，github的并未在其中，因此在连接github时ssh agent在这个配置文件中找不到连接github的信息，因此出现错误。那么我们可以使用 `ssh-add ~/.ssh/<your_private_key_file>` 来将密钥文件加载进缓存中。你可以使用 `ssh-add -l` 来查看是否成功将密钥加入缓存。

