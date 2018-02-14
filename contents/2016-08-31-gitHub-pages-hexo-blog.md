这篇博客是使用**Github Pages + Hexo** 搭建本博客的总结。

作为一个有多年开发经验的人来说，没有搭建自己的博客，记录自己的开发点点滴滴，实在有点遗憾，以前总以为搭建博客得买服务器，域名等复杂的步骤，然后选择了国内老牌的博客CSDN或博客园来写博客，但可拓展性太低，千篇一律。

最近经过朋友的介绍使用GitHub Pages + Hexo搭建免费的独立博客，痛下决心来做一次。

## 前言
搭建博客需要先了解几个知识点
* [Git](https://git-scm.com/book/zh/v2)
* [GitHub](https://github.com/)
* [GitHub Pages](https://pages.github.com/)
* [Hexo](https://hexo.io/zh-cn/)
* [Markdown](http://www.appinn.com/markdown/#overview)

## 环境配置
### 安装Hexo
由于安装Hexo需要Node.js和Git，所以在[Hexo](https://hexo.io/zh-cn/)官网上有安装的详细说明（中文版），不再赘述（Mac电脑有自带Git的）。

当Node.js和Git都安装完成后，开始正式安装Hexo，在终端上执行
``` bash
$ sudo npm install -g hexo
```
需要输入管理员密码（Mac登录密码）。(`sudo`：Linux管理员指令，`-g`：全局安装)
> 坑一：[Hexo官网](https://hexo.io/zh-cn/)上的安装命令是`$ npm install -g hexo-cli`，安装时不要忘记前面加上`sudo`，否则会因为权限问题报错。

### 初始化博客
终端cd到你想创建博客的文件夹，然后中执行终端命令
```bash
$ hexo init blog
```
blog是你新建的博客文件夹，cd到这个文件夹，然后执行命令安装npm

```bash
$ npm install
```
最后执行命令，开启Hexo本地服务器

```bash
$ hexo s // 全称：hexo server
```
终端中会提示在浏览器打开 http://localhost:4000 这个网址就可以看到（**因为我配置了主题内容，请忽略图片上的文字，之后将会讲到主题**）

![Snip20160831_1](http://p44bkxib3.bkt.clouddn.com/Snip20160831_1.png)

## 关联Github
### 创建仓库
在前言的[GitHub Pages](https://pages.github.com/)中介绍了Pages仓库，所以要创建一个名为`用户名.github.io`固定写法的参考，例如`piglikeyoung.github.io`，如下图所示

![Snip20160831_2](http://p44bkxib3.bkt.clouddn.com/Snip20160831_2.png)

本地的blog目录结构

```
.
├── _config.yml
├── package.json
├── scaffolds
├── source
|   ├── _drafts
|   └── _posts
└── themes
```

用文本编辑器打开`_config.yml`
修改下列代码

```
	 deploy:
    type: git
    repository: https://github.com/piglikeYoung/piglikeyoung.github.io.git
    branch: master
```
你需要把`repository`替换成你自己的github地址
> 坑二：在配置所有的`_config.yml`文件时（包括theme中的），在所有的`冒号:`后边都要加一个`空格`，否则执行hexo命令会报错，切记 切记

在 blog文件夹目录下执行下面命令生成静态页面
```bash
$ hexo g        //全称：hexo generate
```

```
如果出现如下报错：
ERROR Local hexo not found in ~/blog
ERROR Try runing: 'npm install hexo --save'
则执行命令：
npm install hexo --save
若无报错，自行忽略此步骤。
```
再执行命令部署静态网页
```bash
$ hexo d            //全称：hexo deploy
```
> 坑三：执行`hexo d`报错无法连接git或找不到git，执行这个命令 
```bash
$ npm install hexo-deployer-git --save
```

再次执行`hexo g`和`hexo d`命令。

如果你的电脑没有管理Github，执行`hexo d`时终端会提示你输入Github的用户名和密码

```bash
Username for 'https://github.com':
Password for 'https://github.com':
```
当`hexo d`执行成功，在浏览器上输入 http://piglikeyoung.github.io 就会显示 http://localhost:4000 一样的页面。

**为了避免每次都输入Github用户名和密码，可以参考下一章来配置SSH Keys**

## 添加SSH Key到Github
### 判断是否已经生成了SSH Key
如果已经生成了SSH Key会在`~/.ssh/id_rsa.pub`目录下面找到`id_rsa`和`id_rsa.pub`这两个文件。
如果没有生成执行下面命令生成
```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```
其中`your_email@example.com`是你的Github邮箱。
在Github上有这个[Help文档](https://help.github.com/articles/generating-an-ssh-key/)指导你如何生成SSH Key，按照步骤一步步来。

最后在Github网页上个人设置找到SSH Key填写界面，把`id_rsa.pub`里面的内容复制粘贴进去就完成了。

![Snip20160901_2](http://p44bkxib3.bkt.clouddn.com/Snip20160901_2.png)

## 发布文章
终端cd到`blog`目录下执行新建文件的指令

```bash
hexo new "postName"
```
`postName`是新建的md文件的名称，会在`/blog/source/_posts`目录下生成一个名为`postName.md`的文件。可以使用`MWeb`编辑器修改文章，支持预览，由于MWeb是收费的，可以使用免费的Mou编辑器。

当文章编辑完成，在blog目录下，执行下面命令

```bash
hexo g //生成静态页面
hexo d //将文章部署到Github
```
**浏览器打开`http://piglikeyoung.github.io`已经能够访问到基于Github的Hexo的博客了。**

> Tips：Hexo支持热更新，所以不用每次修改了一点文章内容都部署到Github上查看变化，你可以本地开启Hexo，一边本地修改，一边在浏览器上刷新 http://localhost:4000/ ，就可以看到刚才的修改。

## 主题安装
Hexo支持很多主题，它有个[主题网址](https://hexo.io/themes/)，在上面能够找到很多主题。

我使用的是[jacman](https://github.com/wuchong/jacman)这个主题。打开jacman网址有个中文说明教程来指导你如何安装主题，在blog目录下执行命令

```bash
git clone https://github.com/wuchong/jacman.git themes/jacman
```

修改博客根目录下的配置文件 `_config.yml`，把`theme`的值修改为 `jacman`。
执行命令重新部署Hexo

```bash
hexo clean //清除缓存文件 (db.json) 和已生成的静态文件 (public)
hexo g //生成静态页面
hexo d //将文章部署到Github
```
**如果要详细配置主题请参考[jacman](https://github.com/wuchong/jacman)上的中文文档，相当详细。**

我个人fork了一份原版的主题，做了些许修改，如果你要做修改可以参考我的fork提交[myJacman](https://github.com/piglikeYoung/jacman)

## 绑定个人域名
有的朋友不太喜欢那个访问域名，因为那个Github提供的二级域名，喜欢个性化的朋友可以购买自己的域名。购买域名有很多途径，比如[GoDaddy](https://sg.godaddy.com/zh/)，[阿里万网](http://wanwang.aliyun.com/)。在国内买域名，好处是网站是中文的方便后期维护，坏处是解析域名大部分要实名认证，上传身份证复印件，国外就没这个要求。

### 主题添加域名
在**/blog/themes/jacman/source**目录下新建一个名为`CNAME`的文件，不带后缀，直接把自己购买的域名`piglikeyoung.com`写入。
重新部署Hexo，就可以实现域名跳转（如果域名已经解析完成）。

### 域名解析
域名购买完成，必须经过解析才能实现跳转，如果在阿里万网上购买，可以直接点击添加解析，如果在国外购买可以使用[DNSPOD](https://www.dnspod.cn/)来解析域名。
参考下图来添加解析

![Snip20160902_4](http://p44bkxib3.bkt.clouddn.com/Snip20160902_4.png)

记录类型：CNAME
主机记录：example.com（不要www），填写@或者不填
记录值：piglikeyoung.github.io. (不要忘记最后的.，piglikeyoung改为你自己的用户名)，点击保存即可

此时，点击 http://piglikeyoung.com/ 已经可以实现跳转，http://piglikeyoung.com/ 和 http://piglikeyoung.github.io. 会显示一样内容。


## 优化部署和管理
通过上面的内容，相信你已经能够顺利的完成Hexo部署，将你的md文件转化为静态网页了。此时也许你会问：
* 当我在自己电脑写博客，到了公司也想用公司的电脑写博客
* 重装电脑了博客文章怎么办

这样你得做好备份了，使用百度云或者U盘等等，但是，你不觉得这不够优雅吗？放着一个Github不用，去用别的备份工具。为此我搜索了很多解决方案。
最简单的就是利用`分支`。

**一个Github Pages仓库，至少创建2个分支，一个是`master`用于部署网站，一个是`hexo`用来存放你的博客文件。**

### 创建步骤
* 创建仓库`piglikeyoung.github.io`
* 创建两个分支：`master`（默认就存在）和`hexo`
* 切换到`hexo`分支，clone到本地
* 在`hexo`分支下，执行`hexo init blog`，`npm install`等初始化博客命令，并且执行git版本控制命令，提交到github
* 修改`_config.yml`里面的deploy参数，设置部署分支为`master`
* 最后执行`hexo g -d`生成静态网页并部署到github上

**总之，在piglikeyoung.github.io仓库就有了两个分支，master用于部署生成的网页，hexo用于管理博客文件。**

### 日常维护
你只需要在hexo的分支下完成博客的增删改查，通过Hexo指令部署到master分支。

当你使用另一台电脑修改博客，你只需这么做：
* 把piglikeyoung.github.io仓库clone到本地，切换到`hexo`分支
* 在clone的本地仓库文件夹下依次执行：`npm install hexo`、`npm install`、`npm install hexo-deployer-git`（切记，不需要执行hexo init这条指令，它会重新创建一个新的博客）。

## 后记
本博客的第一篇文章总算写完了，其中参考很多网上优秀博文才能顺利搭建完成，非常感谢他们的无私贡献。
* [Mac上搭建基于GitHub的Hexo博客](http://gonghonglou.com/2016/02/03/firstblog/)
* [从 Octopress 迁移到 Hexo](http://blog.devtang.com/2016/02/16/from-octopress-to-hexo/)






