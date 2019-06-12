---
title: 先装ubuntu，再安装windows系统后，如何安装grub
date: 2019-03-26 18:29:28
tags:
  - windows
  - ubuntu
  - 操作系统
---

先装ubuntu，再安装windows系统后，如何安装grub接管开机启动。

<!-- more -->

网上推荐的安装双系统的方法，都是说要先装windows，再装ubuntu，这样最后可以用grub接管开机启动，在启动的时候可以选择进入ubuntu还是windows。

但是我安装的顺序是先ubuntu，再windows。windows安装的时候清除了grub，这样我每次想要换系统都得进入BIOS切换启动顺序，很不方便。之后我查阅了ubuntu的相关文档<https://help.ubuntu.com/community/RecoveringUbuntuAfterInstallingWindows>，从而解决了这个问题。

我只采用了图形方式，命令行方式的我没有尝试。

操作步骤很简单，就是首先安装并启动`boot-repair`

```bash
sudo add-apt-repository ppa:yannubuntu/boot-repair
sudo apt-get update
sudo apt-get install -y boot-repair && boot-repair
```

然后点击`Recommended repair`，等待完毕后重启即可。