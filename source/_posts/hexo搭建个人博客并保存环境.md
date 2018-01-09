---
title: hexo搭建个人博客并保存环境
date: 2018-01-04 14:29:01
tags: [hexo]
---

## hexo搭建个人博客并保存环境
搭建过程可以参考网上一篇教程：[Mac上搭建hexo](https://www.jianshu.com/p/17d32db14aa6)

按照以上博客的配置，即可将个人博客搭建好，如果你想要将博客的环境保存下来，下次换台机器后还能继续使用之前配置的环境的话，可以在blog文件夹下新建一个git仓库(git init)，将该仓库托管到github上的某一个仓库里（我是将blog文件夹下的全部文件托管在github中博客所在仓库的hexo分支上），每次重新搭建环境的时候只需要从该分支拉取下来所有的环境信息即可在原来的基础上开始写新的博客。

在执行完：```node install -g hexo```后，不要执行```hexo init```，进入到blog文件夹，执行`git init`，创建git仓库，执行`git remote add origin git@github.com:konnase/konnase.github.io.git`添加原来的环境所在的远程git仓库，执行`git pull origin hexo:master`将远程的仓库拉取到本地，执行`npm install`安装hexo相关环境。到此，即可无缝使用原来的hexo环境了。