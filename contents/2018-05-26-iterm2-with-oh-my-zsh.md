
## iTerm2 + Oh My Zsh 打造舒适终端体验

最近尝试用终端来代替图形界面，更能了解本质，使用了 **iTerm2** 来打造了一个相对友好的终端使用，看一下最终效果：

![WX20180609-151847@2x](http://p44bkxib3.bkt.clouddn.com/WX20180609-151847@2x.png)

### 下载iTerm2

直接去官网下载 [iTerm2](https://www.iterm2.com/)

安装完成后，在/bin目录下会多出一个zsh的文件。

Mac系统默认使用dash作为终端，可以使用命令修改默认使用zsh：

```shell
chsh -s /bin/zsh
```

如果想修改回默认dash，同样使用chsh命令即可：

```shell
chsh -s /bin/bash
```
更多命令查询参考链接

### 安装Oh my zsh

安装方法有两种，可以使用curl或wget，看自己环境或喜好：

```shell
# curl 安装方式
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

```shell
# wget 安装方式
sh -c "$(wget https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh -O -)"
```

oh-my-zsh开源地址：https://github.com/robbyrussell/oh-my-zsh

安转好oh-my-zsh后，主题文件夹在 **~/.oh-my-zsh/themes**

修改 **~/.zshrc** 配置文件 **ZSH_THEME="agnoster"**

这个主题比较好看，看个人喜好了。

### 安装PowerLine

powerline官网：http://powerline.readthedocs.io/en/latest/installation.html

安装powerline的方式依然简单，也只需要一条命令：

```shell
pip install powerline-status --user
```

没有安装pip的同学可能会碰到zsh: command not found: pip。

使用命令安装pip即可：

```shell
sudo easy_install pip
```

### 安装PowerFonts

安装字体库需要首先将项目git clone至本地，然后执行源码中的install.sh。

下载地址 https://github.com/powerline/fonts

进入文件夹执行命令：

```shell
./install.sh
```

安装好字体库之后，我们来设置iTerm2的字体，具体的操作是iTerm2 -> Preferences -> Profiles -> Text，在Font区域选中Change Font，然后找到Meslo LG字体。有L、M、S可选，看个人喜好：
![fonts](http://p44bkxib3.bkt.clouddn.com/fonts.png)

我这里设置的字体是：**14pt Meslo LG S DZ Regular for Powerline**

## 配色

最终配色还是需要自己喜好去配置，详细配置步骤可以去参考链接。

## 参考链接

[chsh命令](http://man.linuxde.net/chsh)
[iTerm 2 && Oh My Zsh【DIY教程——亲身体验过程】](https://www.jianshu.com/p/7de00c73a2bb)
[iterm2-with-oh-my-zsh](https://github.com/sirius1024/iterm2-with-oh-my-zsh)

