
## 前言

众所周知，现在APP迭代的速度非常快，对开发和测试都是严重的考验，有很多客观因素可以通过自动化集成来减少人力成本，增加开发效率。`fastlane` 就是一个可以提高打包效率的工具。

## fastlane

fastlane 如何安装，怎么使用，可以Google或者baidu，也可以访问[官网](https://docs.fastlane.tools/getting-started/android/setup/)，总的来说它是一套自动化打包的工具集，用 Ruby 写的，用于 iOS 和 Android 的自动化打包的发布等工作。gym是其中的打包命令。

fastlane 主要有以下指令：

* deliver: 上传屏幕截图、二进制程序数据和应用程序到App Store
* snapshot: 自动截取你的程序在每个设备上的图片
* frameit: 快速将截图放入对应的手机设备中
* pem: 自动生成和更新应用推送通知描述文件
* sigh: 生成配置文件
* produce: 通过命令行在iTunes Connect创建一个新的iOS app
* cert: 自动创建iOS证书
* gym: 建立新的发布的版本，打包

一个最完整的发布过程使用fastlane可以这样描述：

```
lane :appstore do
  increment_build_number
  cocoapods
  xctool
  snapshot
  sigh
  deliver
  frameit
  sh "./customScript.sh"
slack end
```

1. 增加 build 的版本号
2. cocoapods进行pod配置
3. xctool进行编译
4. snapshot自动生成截图
5. sigh处理配置文件
6. deliver上传截图
7. frameit将应用截图快速放入对应的设备尺寸中
8. 执行些自动化脚本

## 实践

现在实践：执行命令-->打包-->上传Bugly

首先，cd到项目的根目录，执行

```shell
fastlane init
```

初始化的过程会让你填写一些项目信息，比如Apple ID，项目的Targets（如果你有多个的话），还会生成如下图的项目结构：
![WX20170514-140038](http://p44bkxib3.bkt.clouddn.com/WX20170514-140038.png)

* Appfile：里存放App基本信息包括app_identifier、apple_id、team_id。

* Fastfile：就是编写执行action的文件，所有的自定义功能都写在这个文件里面。

我使用的脚本如下：

```shell
ios_scheme_name = "Livestar.tv"
ios_ipa_name = "Livestar"
debug_ipa_path = "/Users/pikeyoung/Documents/" + Time.now.strftime("%Y-%m-%d") + "/Debug/"
release_ipa_path = "/Users/pikeyoung/Documents/" + Time.now.strftime("%Y-%m-%d") + "/Release/"

before_all do
  # git_pull(only_tags: true)
  cocoapods
end

after_all do
  # push_git_tags
end

lane :createDebugIPA do
  ipa_path = debug_ipa_path
  ipa_name = ios_ipa_name
  ios_app_version = get_info_plist_value(path: "./" + ios_scheme_name + "/Info.plist", key: "CFBundleShortVersionString")
  ios_app_build = get_info_plist_value(path: "./" + ios_scheme_name + "/Info.plist", key: "CFBundleVersion")
  gym(
    scheme: ios_scheme_name,
    output_name: ipa_name + "_" + ios_app_build, # 输出的IPA名称
    silent: true, # 隐藏不必要的信息
    clean: true, # 在构建前先clean
    configuration: "Debug", # 指定要打包的配置名
    export_method: 'ad-hoc', # 指定打包所使用的输出方式，目前支持app-store, package, ad-hoc, enterprise, development, 和developer-id，即xcodebuild的method参数
    output_directory: ipa_path # IPA输出目录
  )
end
```

上面就是在指定目录下创建一个Debug的IPA，由于我司没有使用企业证书，构建的流程比较简单，所以没有使用很复杂的打包脚本。fastlane 支持从外部传入参数，指定打包的环境，增加build的版本号等action，目前都没用上。

上传Bugly就很简单了，需要在Bugly官网下载upload的Ruby文件：
![WX20170514-142857](http://p44bkxib3.bkt.clouddn.com/WX20170514-142857.png)

然后还是通过fastlane执行action：

```shell
lane :uploadDebugIPA do
  ios_app_version = get_info_plist_value(path: "./" + ios_scheme_name + "/Info.plist", key: "CFBundleShortVersionString")
  ios_app_build = get_info_plist_value(path: "./" + ios_scheme_name + "/Info.plist", key: "CFBundleVersion")
  debug_ipa_upload_path = debug_ipa_path + ios_ipa_name + "_" + ios_app_build + ".ipa"
  upload_app_to_bugly(
    file_path: debug_ipa_upload_path,
    app_key: "123456",
    app_id: "456123",
    pid: "2",
    title: "iOS-Debug-" + Time.now.strftime("%Y-%m-%d %H:%M:%S"),
    desc: "内部测试,请勿外泄"
  )
end
```

假如，你需要从外部传入参数，确认需要打包的git分支，你可以这样：

```shell
lane :ci do|options|
    branch = options[:branch] # 获取传入的git分支名
    build_no = get_version_number + '.' + Time.new.strftime("%m%d%H%M") # 生成build号，获取版本号+时间

    puts "Begin to run ci"

    # 确认分支、git状态、拉取最新代码
    sh "git checkout #{branch}"
    ensure_git_branch(branch: branch)
    ensure_git_status_clean
    git_pull

    # 递增build number
    increment_build_number(build_number: build_no)

    #开始打包
    gym(
      export_method:"development",
      output_directory:"./fastlane/build",
    )

    # 使用fir-cli上传ipa
    sh "fir publish ./build/LSTestDemo.ipa -T fasdfsdafas13213213sfs"

end
```

然后这样执行：

```shell
fastlane ci branch:dev1
```

## 总结

上面的action还是比较简单的，上面的步骤其实还可以优化下，使用一台专门的Mac电脑配合`Jenkins`做打包操作。

一些常用的 fastlane action 指令总结：

```shell
git_pull: git拉取代码
cocoapods: 更新pod库
push_git_tags: git推送tags
Time.now.strftime("%Y-%m-%d"): 获取现在的时间，格式化显示格式
Time.new.strftime("%m%d%H%M"): 获取现在的时间，格式化显示格式（new与now的区别在于，new会调用initialize.）
get_version_number: 获取项目version number
ensure_git_branch(branch: branch): 确认git分支
ensure_git_status_clean: 检查git状态
increment_build_number: 递增build version number
push_to_git_remote: git推送代码到远程仓库
```
最后推荐下官方给的一些例子，是国外很多优秀的例子，可以直接借鉴过来：
https://github.com/fastlane/examples

## 后记

附上我自己使用的打包脚本：

```shell
#!/bin/bash

#设置超时
export FASTLANE_XCODEBUILD_SETTINGS_TIMEOUT=120

#计时
SECONDS=0

#假设脚本放置在与项目相同的路径下
project_path=$(pwd)
#输出文件目录
ipa_output_path="/Users/pikeyoung/Documents/"
#取当前时间字符串添加到文件结尾
now=$(date +"%Y_%m_%d_%H_%M_%S")

#指定项目的scheme名称
scheme="PocketCrane"
#指定要打包的配置名
configuration="Debug"
#指定打包所使用的输出方式，目前支持app-store, package, ad-hoc, enterprise, development, 和developer-id，即xcodebuild的method参数
export_method='ad-hoc'

#指定项目地址
workspace_path="$project_path/PocketCrane.xcworkspace"
#指定输出路径
output_path="$ipa_output_path/Debug"
#指定输出归档文件地址
archive_path="$output_path/PocketCrane_${now}.xcarchive"
#指定输出ipa地址
ipa_path="$output_path/PocketCrane_${now}.ipa"
#指定输出ipa名称
ipa_name="PocketCrane_${now}.ipa"
#获取执行命令时的commit message
commit_msg="$1"

#输出设定的变量值
echo "===workspace path: ${workspace_path}==="
echo "===archive path: ${archive_path}==="
echo "===ipa path: ${ipa_path}==="
echo "===export method: ${export_method}==="
echo "===commit msg: $1==="

#先清空前一次build
fastlane gym --workspace ${workspace_path} --scheme ${scheme} --include_bitcode false --silent true --clean false --configuration ${configuration} --archive_path ${archive_path} --export_method ${export_method} --output_directory ${output_path} --output_name ${ipa_name}

#上传到fir
fir publish ${ipa_path} -T "123456" -c "${commit_msg}"

#输出总用时
echo "===Finished. Total time: ${SECONDS}s==="
```

## 参考链接

* [fastlane docs](https://docs.fastlane.tools/)
* [iOS菜鸟福利！带你一键轻松搞定项目构建、封包、上传](http://www.jianshu.com/p/38786933ba9c)
* [记fastlane一次实践](http://www.jianshu.com/p/8e571c835844?utm_campaign=maleskine&utm_content=note&utm_medium=writer_share&utm_source=weibo)
* [手把手教你利用Jenkins持续集成iOS项目](http://www.jianshu.com/p/41ecb06ae95f#)
* [Jenkins一键发布「apk&ipa」 到Bugly](http://blog.csdn.net/zhf198909/article/details/53365812)
* [fir-cli](https://github.com/FIRHQ/fir-cli)
* [iOS可持续化集成: Jenkins+bundler+cocoapods+fastlane](http://www.cocoachina.com/ios/20150728/12733.html)
* [fastlane Tutorial: Getting Started](https://www.raywenderlich.com/136168/fastlane-tutorial-getting-started-2)
* [Fastlane入门:初级使用篇](http://www.jianshu.com/p/9f66b7a106ea)
* [Fastlane 入门实战教程](https://libraries.io/github/mythkiven/AD_Fastlane)




