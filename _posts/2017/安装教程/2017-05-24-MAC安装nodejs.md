---
layout: post
title: MAC安装nodejs
category: 安装教程
tags: [mac,nodejs]
excerpt: MAC安装nodejs
keywords: MAC,nodejs,安装教程
---


# mac安装nodejs


mmp安装过程真的折腾死我了。

先安装brew


然后下载node.js

```shell script
brew install node
```

下载完成之后，验证下版本：

```shell script
node -v
npm -v
```

然后再下载淘宝镜像

```shell script
npm install -g cnpm --registry=https://registry.npm.taobao.org
```

权限不够的话，加上sudo

```shell script
sudo npm install -g cnpm --registry=https://registry.npm.taobao.org
```

到此就已经安装完成了，开始你的nodejs吧


