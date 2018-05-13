---
title: 利用github分支同步博客代码，方便不同机器上更新博客
date: 2018-05-13 14:51:29
tags: [hexo,github]
---
使用github搭建博客后，每次更新需要在原来的代码基础上更新。如果在其他个人电脑上，没有原来的代码，则无法立即同步更新博客。这时候我们可以选择将代码push到一些代码服务器上，但是我们这里可以这么做。<!-- more -->  
首先，我们先对github中博客的代码仓库建立一个新的分支hexo_code，并设置代码库的Branches->Default branches为新建的分支hexo_code。  
克隆代码库到本地:  
```
git clone git@github.com:Users/page.github.com.git
cd page.github.com.git
```
将博客代码全部拷贝到page.github.com.git目录, 如果themes不是使用的默认的theme。则需要删除掉该主题目录中的.git目录。  
这时我们就可以把当前目录全部推送到github的新branches了。  
```
git add .
git commit -m 'new'
git push origin hexo_code
```
这样我们在其他的电脑上，只需要clone代码(记得添加ssh key)，并在目录中执行npm install。之后就能按照正常的操作更新博客了。  
```
git clone git@github.com:Users/page.github.com.git
cd page.github.com.git
#npm install #在其他机器上执行

# 更新博客
hexo new 'test' # 新建文章
hexo g          # 生成页面代码
hexo d          # 部署页面

# 同步博客代码到git
git add source/_posts/test.md
git commit -m "new up test"
git push origin hexo_code
```
done.  