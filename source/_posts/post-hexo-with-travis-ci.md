---
title: 持续集成发布 Hexo
date: 2018-02-03 17:58:29
updated: 2018-02-03 22:36:29
tags: [hexo,CI]
categories: Blog
---

![travis logo](http://www.ruanyifeng.com/blogimg/asset/2017/bg2017121901.png)

> 每次写完新文章后，都需要手动推送到[Gayhub](https://github.com/)的分支上，这样的重复性操作很容易让人厌烦，而且博客源码和博客本身是分离的，只有博客才有版本控制，所以如果身边没有博客源码，那么就不能发布博客了。基于以上的因素，决定采用[Travis CI](https://travis-ci.org/)对博客使用持续集成的方式发布文章。

在真正开始之前，需要在博客仓库上新建一个 hexo 分支，存储博客源码，master 分支则继续存储生成的静态博客。

<!--more-->

#### 创建 Github Token

新建一个 token，选择 repo 权限，这个会在下面设置 travis 时用到，注意，token 只出现一次，需要妥善保存

![](https://s1.ax1x.com/2018/02/03/9eBcXd.png)

#### 设置 Travis CI

使用 github 帐号登录[Travis CI](https://travis-ci.org/)，开启博客仓库并设置启动 CI 的触发条件

![step 1](https://s1.ax1x.com/2018/02/03/9eBCwt.png)

![setup 2](https://s1.ax1x.com/2018/02/03/9eBPTP.png)

* Build only if .travis.yml is present：是只有在.travis.yml 文件中配置的分支改变了才构建
* Build pushes：当推送完这个分支后开始构建
* Environment Variables：新建一个变量，value 填写上面申请的**token**，name 作为变量名，在.travis.yml 中用到

#### 编写.travis.yml 文件

在 hexo 分支根目录下新建一个.travis.yml 文件，具体内容可参考下文：

```yaml
language: node_js # 设置语言
node_js: stable # 设置相应版本
cache:
    apt: true
    directories:
        - node_modules
before_install:
    - export TZ='Asia/Shanghai'
    - npm install hexo-cli -g
    - chmod +x ./publish-to-gh-pages.sh
install:
    - npm install
script:
    - hexo clean
    - hexo g
after_script:
    - ./publish-to-gh-pages.sh
branches:
    only:
        - hexo #只监测 hexo 源码分支的触发条件，
env:
    global:
        - GH_REF: github.com/xuexiaoao/xuexiaoao.github.io.git #设置 GH_REF，注意更改
```

这个文件其实就是包含命令的脚本，按照 before_install--->install--->script--->after_script 的顺序执行相应命令，在 after_script 阶段调用**publish-to-gh-pages**脚本执行推送到 master 分支的任务。

```shell	
#!/bin/bash
set -ev
# master start
git clone https://${GH_REF} .deploy_git
cd .deploy_git
git checkout master
cd ../
mv .deploy_git/.git/ ./public/
## master end
cd ./public
git config user.name "xuexiaoao"
git config user.email "xuexiaoaoooo@gmail.com"
# add commit timestamp
git add .
git commit -m "Travis CI Auto Builder at `date +"%Y-%m-%d %H:%M"`"
git push --force --quiet "https://${Travis_Token}@${GH_REF}" master:master #Travis_Token是上文新建的变量名，GH_REF则是在.travis.yml最后定义的变量
```

需要注意的是，如果仅仅只是执行**git push**命令，会丢失以前所有的提交信息，所以为了不丢失这些信息，需要执行**master start**到**master end**的内容。

#### 结束

写完文章就可以在 hexo 分支下执行 git push 命令，打开[travis](https://travis-ci.org)，如果发现和下图一样的 passed，就说明持续集成发布博客已经大功告成了。

![build pass](https://s1.ax1x.com/2018/02/03/9ecmXF.png)