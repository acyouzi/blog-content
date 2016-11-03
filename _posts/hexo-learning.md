---
title: hexo 快速入门
date: 2016-11-01 19:53:07
author: "acyouzi"
cdn: header-off
header-img: img/dog.jpg
tags:
	- 编程
	- hexo
---

### 安装
建议新手在 linux 下开发，可以免去很多乱七八糟的问题
一般linux都默认安装有git 但是不会默认带有nodejs,所以我们首先要安装nodejs
下载nodejs并解压到/opt目录

    wget https://nodejs.org/dist/v6.9.1/node-v6.9.1-linux-x64.tar.xz
    tar -xvf node-v6.9.1-linux-x64.tar.xz -C /opt

配置环境变量

    echo "export NODE_HOME='/opt/node-v6.9.1-linux-x64'" >> /etc/profile
    echo "export PATH=$PATH:$NODE_HOME/bin" >> /etc/profile
    source /etc/profile

测试一下nodejs有没有问题

    node -v

安装 hexo

    npm install -g hexo-cli

### 新建项目

    cd ~ && mkdir my-blog && cd my-blog
    hexo init ./
    npm install

现在应该能看到这样的目录结构

    .
    ├── _config.yml
    ├── package.json
    ├── scaffolds
    ├── source
    |   ├── _drafts
    |   └── _posts
    └── themes

下面简单配置一下 _config.yml 的内容，官网给出了比较详细的说明，这里就不在赘述了
> https://hexo.io/zh-cn/docs/configuration.html

下面启动一下服务器看看页面效果,如果时第一次使用需要先安装 hexo server

    npm install hexo-server --save
    hexo generate
    hexo server

服务器监听 localhost:4000，现在可以访问一下看看了。

### 更换主题
hexo 能够非常方便的更换主题，首先我们需要找到一些漂亮的主题，可以从github上搜索。
这里我们随便选一款主题作为例子

    cd ~/my-blog/themes/
    git clone https://github.com/Haojen/hexo-theme-Anisina.git

配置 _config.yml 文件

    theme: hexo-theme-Anisina

然后重新 generate 一下，看下效果

    hexo clean && hexo generate
    hexo server

### 写文章
如果文章中需要插入一些资源，可以在 _config.yml 把如下配置项设置为true

    post_asset_folder: true

这样在通过命令行创建文章时，会在同目录下生成一个同名文件夹，可以把本页面用到的资源放到这里面

    hexo new [layout] <title>

然后就可以去source/_posts文件夹下编辑你的文章了。

### 发布到github
首先需要创建一个gitpage仓库，可以参考官网的教程

    https://pages.github.com/

1. 首先 创建一个以 your-username.github.io 命名的仓库
2. 修改 _config.yml 文件的 deploy配置

        # Docs: https://hexo.io/docs/deployment.html
        deploy:
          type: git
          repo: git@github.com:xxx/xxx.github.io.git
          branch: master

3. 生成和发布

        hexo generate
        npm install hexo-deployer-git --save 
        hexo deploy

4. 访问地址 xxx.github.io 看看是不是能看到你发布的网站 ^-^

