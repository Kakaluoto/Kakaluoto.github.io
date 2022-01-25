---
title: 【博客】Hexo在多台电脑上提交和更新
date: 2022-1-10
tags: [hexo,Git]
cover: https://s4.ax1x.com/2022/01/10/7VEnLd.jpg
mathjax: true
---

## 前言

我现在有两台电脑，最初第一次装好hexo环境的电脑在宿舍，姑且叫这台电脑**"老电脑"**吧，代表最初拥有hexo环境的电脑，然后事情是这样的去到工位的电脑上想要更新博客总是要远程启动宿舍的电脑才行，于是想要在工位电脑也能更新，这里工位的电脑姑且叫做**"新电脑"**吧。

最初搞这个多设备同步属实折腾了好半天，看了很多博客也在知乎上参考了不少，但总是需要在不同博客之间相互参考最终才完美解决，所以想要把这几天的经历总结一下。

## 一. hexo同步原理
### 1.  hexo博客目录结构说明

这是老电脑上的目录结构

![](https://s4.ax1x.com/2022/01/10/7EKIeg.png)

| 文件夹        | 说明                                                         | 是否需要上传github |
| :------------ | ------------------------------------------------------------ | :----------------- |
| node_modules  | hexo需要的模块，就是一些基础的npm安装模块，比如一些美化插件，在执行`npm install`的时候会重新生成 | 不需要             |
| themes        | 主题文件                                                     | 需要               |
| public        | hexo g命令执行后生成的静态页面文件                           | 不需要             |
| packages.json | 记录了hexo需要的包的信息，之后换电脑了npm根据这个信息来安装hexo环境 | 需要               |
| _config.yml   | 全局配置文件，这个不用多说了吧                               | 需要               |
| .gitignore    | hexo生成的默认的.gitignore模块                               | 需要               |
| scaffolds     | 文章的模板                                                   | 需要               |
| .deploy_git   | hexo g自动生成的                                             | 不需要             |

### 2. 同步原理

主要思路是利用git分支来实现hexo的同步。

hexo生成的静态页面文件默认放在master分支上，这是由_config.yml配置文件所决定的

你可以在全局配置文件_config.yml中找到这么一段

```yaml
# Deployment
## Docs: https://hexo.io/docs/one-command-deployment
deploy:
  type: git
  repo: git@github.com:username/username.github.io.git
  branch: master
```

因此每当我们执行hexo d的时候，hexo都会帮我们把生成好的静态页面文件推到master分支上。

(我这里可能和你的不太一样，因为之前使用http推送经常由于网络问题推送上不去，所以我复制了username.github.io仓库的ssh链接，用ssh推送成功率更搞，所以我的链接形式是git@github.com:username/username.github.io.git这样的)

在我们第一次部署好博客的时候，github给我们创建的唯一一个分支就是master分支，同时也是默认分支。默认分支就意味着每次我们执行`git clone 仓库地址 `或者`git pull 仓库地址`拉取的是默认分支的代码。

但是执行hexo d 对应的分支和默认分支是没有关系的，因为这是由配置文件决定的，配置文件写的哪个分支就是哪个分支。

因此，hexo生成的静态博客文件默认放在master分支上。hexo的源文件（部署环境文件）可以都放在source分支上（可以新创建一个source分支）。然后把source分支设置成默认分支。有小伙伴可能会担心默认分支的改变会不会影响到原来的网页的正常显示，其实如果是用GitHub Pages对博客进行托管的话也很简单，第一次搭建博客默认使用master分支作为页面。在下图所示的设置里可以找到。如果不小心搞错了只要把分支设置成静态页面对应的分支就好了。

![](https://s4.ax1x.com/2022/01/10/7EYj6s.png)

把source分支设置成默认分支，用来存放源文件，master分支依然存放静态文件。在**老电脑**上，我们需要把必要的源文件push到source分支。换**新电脑**时，直接`git clone 仓库地址 `此时会从source分支下载源文件，剩下的就是安装hexo环境，在**新电脑**上就可以重新生成静态页面了，并且因为配置文件clone下来，deploy配置依旧是master分支，所以在**新电脑**上执行`hexo d`还是会把更新过后的静态文件推送到master分支上。

由于master分支和source分支实际上是相互独立的两个普通的分支，所以我们源文件和静态页面的更新也是相互独立的，故而需要手动分别执行`git add . git commit git push`来更新源文件,然后执行`hexo d`更新静态页面。

## 二. 老电脑上的具体操作

这里直接引用我看的一篇博客的步骤,

[链接]: https://www.cnblogs.com/wy0526/p/13066869.html

**1.github准备**

先创建一个分支hexo

![img](https://s4.ax1x.com/2022/01/10/7VVsAI.jpg)

将其设置为默认分支

![img](https://s4.ax1x.com/2022/01/10/7VZpU1.jpg)

**2.打包将要推送到GitHub上的原始文件**

（1）clone该仓库到本地（clone的是hexo默认分支）

```
git clone git@github.com:username/username.github.io.git
```

（2）下载的文件夹里仅留下.git 文件夹，其他的文件都删除

（3）找见我们hexo原位置，将hexo文件夹内除.deploy_git 以外都复制到clone下来的文件夹中

注意：1.现在clone下来的文件夹内应该有个`.gitignore文件`，用来忽略一些不需要的文件，表示这些类型文件不需要git。如果没有，右键新建，内容如下：

```sh
.DS_Store
Thumbs.db
db.json
*.log
node_modules/
public/
.deploy*/
```


2.如果已经clone过主题文件，那么需要把theme主题文件夹里的 .git 也删除。因为git不能嵌套上传，最好是显示隐藏文件，检查一下有没有，否则上传的时候会出错，导致你的主题文件无法上传，这样你的配置在别的电脑上就用不了了。

**3.将clone并修改以后的文件夹推送到远程库**

```
git add .
git commit –m add_branch
git push
```

此时已经成功将整个网站推送到了远程库的默认分支hexo

**补充：后续写文章、修改配置后的保存推送操作**

至此，网站部署至master分支，整个网站备份至hexo分支。当网站的配置或文章修改后都要将远程仓库更新。首先，依次执行

```
git add .``git commit -m ChangeFiles（更新信息内容可改)``git push （或者git push origin hexo)
```

保证hexo分支版本最新。然后执行

```
hexo d -g
```

（在此之前，有时可能需要执行`hexo clean`），完成后就会发现，最新改动已经更新到master分支了，两个分支互不干扰！

## 二. 新电脑上的操作

- 将新电脑的生成的ssh key添加到GitHub账户上
- 在新电脑上克隆username.github.io仓库的source分支(就是存放源码的分支)到本地，此时本地git仓库处于source分支,可以执行`git branch -v`查看。
- 在新电脑的username.github.io文件夹下执行`npm install hexo`、`npm install`、`npm install hexo-deployer-git`（记得，不需要hexo init这条指令）
- 最后执行`hexo g`、`hexo s`、`hexo d`等命令即可提交成功

上面步骤中，`npm install`其实就是读取了packages.json里面的信息，自动安装依赖，有的小伙伴可能只执行`npm install`就行了，不过按照上面的三步是最稳妥的。

这里提一嘴，当新电脑上的操作成功之后，其实对于新电脑还是老电脑其实都无所谓了，任何一台电脑包括老电脑只要安装了NodeJs环境，就都可以按照在新电脑上的操作完整地复刻出一个hexo环境，你甚至可以把老电脑原有的hexo工程删掉再执行上面这几步一样可以快速构建hexo环境，可以看到步骤非常简单。

此外，为了保证同步，推荐先`git pull`合并更新再进行博客的编写。



## 参考链接

[利用Hexo在多台电脑上提交和更新github pages博客 - 简书 (jianshu.com)](https://www.jianshu.com/p/0b1fccce74e0)

[hexo源码上传到GitHub-以防多台电脑操作/重装系统/要将hexo移动到其他磁盘 - 巧莓兔 - 博客园 (cnblogs.com)](https://www.cnblogs.com/wy0526/p/13066869.html)

[Hexo博客备份 - 简书 (jianshu.com)](https://www.jianshu.com/p/57b5a384f234)

[使用hexo，如果换了电脑怎么更新博客？ - 知乎 (zhihu.com)](https://www.zhihu.com/question/21193762)
