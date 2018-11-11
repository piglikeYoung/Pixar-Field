
## Mac 上使用 Charles

## 前言

开发网站的时候可以通过Debug模式查看网络请求状况，在做移动开发的时候通常用到 **Charles**，本文对我日常使用的功能进行总结。

## 介绍

**Charles** 是一个 HTTP 代理服务器，HTTP 监视器,反转代理服务器，当程序连接 Charles 的代理访问互联网时，Charles 可以监控这个程序发送和接收的所有数据。它允许一个开发者查看所有连接互联网的HTTP通信，这些包括 request、 response 和 HTTP headers（包含 cookies 与 cache 信息）。

1. 支持SSL代理。可以截取分析SSL的请求。

2. 支持流量控制。可以模拟慢速网络以及等待时间（latency）较长的请求。

3. 支持AJAX调试。可以自动将json或xml数据格式化，方便查看。

4. 支持AMF调试。可以将Flash Remoting 或 Flex Remoting信息格式化，方便查看。

5. 支持重发网络请求，方便后端调试。

6. 支持修改网络请求参数。

7. 支持网络请求的截获并动态修改。

8. 检查HTML，CSS和RSS内容是否符合W3C标准。

### 常用功能

#### 将Charles设置成系统代理

Charles 提供了两种查看请求的视图：

1. Structure 将网络请求按访问的域名分类

2. Sequence 将网络请求按访问的时间排序

![WX20180902-122021@2x](http://p44bkxib3.bkt.clouddn.com/WX20180902-122021@2x.png)

Charles 是通过将自己设置成代理服务器来完成抓包的，勾选系统代理后，系统本地发出去的请求都能被截取下来。如果只抓取APP的包的话，可关闭此配置，这样不会出现太多的数据。

![WX20180902-122336@2x](http://p44bkxib3.bkt.clouddn.com/WX20180902-122336@2x.png)

> 需要注意的是，Chrome 和 Firefox 浏览器默认并不使用系统的代理服务器设置，而 Charles 是通过将自己设置成代理服务器来完成封包截取的，所以在默认情况下无法截取 Chrome 和 Firefox 浏览器的网络通讯内容。如果你需要截取的话，在 Chrome 中设置成使用系统的代理服务器设置即可，或者直接将代理服务器设置成 127.0.0.1:8888 也可达到相同效果。

如果使用了 **Shadowsocks** 代理，你需要做以下几步操作：

* **Chrome** 安装插件 `proxy-switchyomega`，配置如图

![WX20180902-123105@2x](http://p44bkxib3.bkt.clouddn.com/WX20180902-123105@2x.png)

* **Charles** 的 **Proxy** 菜单下勾选 `External Proxy Settings`，将 Charles 的代理设置为 Shadowsocks 的 HTTP 监听代理地址和端口。

![WX20180902-123314@2x](http://p44bkxib3.bkt.clouddn.com/WX20180902-123314@2x.png)

![WX20180902-123400@2x](http://p44bkxib3.bkt.clouddn.com/WX20180902-123400@2x.png)


#### 截取移动设备上的网络请求包

我们在调试移动APP时，需要抓取APP发送的数据包，首先进行设置，Proxy -> Proxy Settings默认端口是8888，根据实际情况可修改。

![WX20180902-124049@2x](http://p44bkxib3.bkt.clouddn.com/WX20180902-124049@2x.png)

查看本机IP地址：Help -> Local IP Addresses

![WX20180902-124124@2x](http://p44bkxib3.bkt.clouddn.com/WX20180902-124124@2x.png)

然后配置手机代理

![WX20180902-124230@2x](http://p44bkxib3.bkt.clouddn.com/WX20180902-124230@2x.png)


打开要调试的APP，请求就会先发送到Charles，然后验证是否允许访问。
当点击允许后，可以在Proxy -> Access Control Settings里看到可以访问此代理服务器列表

![WX20180902-124506@2x](http://p44bkxib3.bkt.clouddn.com/WX20180902-124506@2x.png)

> 如果不小心点击了拒绝，可以手动添加手机IP/Mac地址到允许访问列表，或者重启Charles，手机再次访问，会再次提示选择。 如果不想每换一个手机都要进行验证，可以配置允许所有手机访问，加入 0.0.0.0/0（IPv4）或 ::/0（IPv6）


##### 手动重复请求（Repeat，Repeat Advanced）

当我们需要对一个请求重复请求的时候，可以对某个请求右击弹出这么这个框，点击重复请求

![WX20180902-130356@2x](http://p44bkxib3.bkt.clouddn.com/WX20180902-130356@2x.png)

##### 手动模拟请求（Compose）

![WX20180902-131039@2x](http://p44bkxib3.bkt.clouddn.com/WX20180902-131039@2x.png)


##### 修改网络请求内容（Backpoints，Compose）

通常我们需要对请求动态修改参数，我们可以通过打断点的方式来修改参数：

![WX20180902-131444@2x](http://p44bkxib3.bkt.clouddn.com/WX20180902-131444@2x.png)


#### 过滤网络请求

从几十个请求里找到我们需要的观察的某个请求比较费时，那么我们就需要对网络请求进行过滤，只监控向指定目录服务器上发送的请求。

有两种方式：

* 在 Sequence 界面的中部的 Filter 栏中填入需要过滤出来的关键字。例如我们的服务器的地址是：*.zayouth.com，那么只需要在 Filter 栏中填入 zayouth 即可。（一般用于临时过滤）

![WX20180902-131549@2x](http://p44bkxib3.bkt.clouddn.com/WX20180902-131549@2x.png)

* 在 Charles 的菜单栏选择 "Proxy"->"Recording Settings"，然后选择 Include 栏，选择添加一个项目，然后填入需要监控的协议，主机地址，端口号。这样就可以只截取目标网站的封包了。如下图所示：（固定过滤地址）

![WX20180902-131926@2x](http://p44bkxib3.bkt.clouddn.com/WX20180902-131926@2x.png)


## 相关链接

* [Charles 从入门到精通](https://blog.devtang.com/2015/11/14/charles-introduction/#%E6%88%AA%E5%8F%96-iPhone-%E4%B8%8A%E7%9A%84%E7%BD%91%E7%BB%9C%E5%B0%81%E5%8C%85)


