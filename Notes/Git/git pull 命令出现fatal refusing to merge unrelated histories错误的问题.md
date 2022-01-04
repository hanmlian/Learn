# git pull 命令出现fatal: refusing to merge unrelated histories错误的问题

## 问题

本地文件上传到gtihub，github上已建立仓库，没有采取git clone先拉取仓库文件，而是直接添加远程仓库提交，在`git init`之后，执行了`git remote add origin + 仓库地址`，设定远程仓库后执行`git add 和 git commit -m`均无问题，执行`git push`时提示如下图

![git-push-error-local](https://cdn.jsdelivr.net/gh/hanmlian/image-hosting@master/Git/git-push-error-local.png)

从报错信息看，一位是本地仓库和远程仓库内容不一致，因为远程仓库有README文件，先执行`git pull origin`命令拉取远程仓库内容（此处图片忘记保存，暂时使用网图）

![git-pull-error-1](https://cdn.jsdelivr.net/gh/hanmlian/image-hosting@master/Git/git-pull-error-1.png)

## 原因

现这个问题的最主要原因还是在于**本地仓库和远程仓库实际上是独立的两个仓库**。假如之前是直接clone的方式在本地建立起远程github仓库的克隆本地仓库就不会有这问题。

## 解决办法

1. pull命令后紧接着使用`--allow-unrelated-histories`选项来解决问题（该选项可以合并两个独立启动仓库的历史）

   ```
   git pull origin --allow-unrelated-histories
   ```

1. 执行命令push到远程仓库

   ```
   git push origin master
   ```
