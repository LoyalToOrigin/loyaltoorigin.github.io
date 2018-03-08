---
title: 关于iOS自动化打包的一些分享
---

说到自动化打包, 相信大家在日常开发中都有所接触, 尤其是在多分支并行开发的情况下, 自动化打包显得尤为重要, 很多时候, 我们打包一般是打及成分支的包, 开发却在开发分支上, 如果采取手动打包, 我们需要反复切分支, 不仅影响工作效率, 而且会打断我们的开发思维, 而却在工程较大的情况下, xcode每次indexing需要的时间就很久。

即使对于很多单分支开发的小项目来说, 自动化打 包的优势也是不言而喻的, 因为在手动打包的同时, 基本可以说是什么事都做不了的, 你需要一步步等待archive, export这些机械化的步骤。而有了自动化打包, 你只需要点击一个按钮, 便可以继续自己的开发。所以, 自动化打包势在必行。

本文主要记录了我在公司自动化打包布置中的一些探索, 及各平台的优缺点和配置过程踩过的坑。

谈到iOS的持续集成, 我们首先想到的一定会是jenkins, 这里我先介绍下我司采用的Mac OS Server(以下简称Server)这个平台的一些优缺点。

## Server相比于jenkins, 我总结优点有三: 

1. 相比于jenkins的各种繁琐配置, Server配置简单, 全程基本下一步操作即可;
2. 直接使用xcode就可开始构建项目, 而不需要登录网页;
3. 集成度相当高, 没有特别的需求, 基本可以不写脚本, 只需要配置一个plist文件即可以打包。
 
这里不做过多的配置介绍, 虽然Server没有jenkins热门, 但网上的文章也比比皆是, 而且如上优点1中所说, Server配置真的很简单, 在证书、描述文件齐全的情况下, 基本就是一直点下一步操作。

下面我介绍使用过程中需要注意的一些方面: 

![](http://p28r7eh75.bkt.clouddn.com/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/xcode_integration.png)

如上图所示, 上图是对bot的各种设置, 一个bot对应一个独立工作空间, 如果有了解jenkins的话, bot可以类比jenkins的一个项目。

如果对打包没有特别需求, 勾选Archive, 选择对应Scheme、Configuration, 指定一个plist文件, 后面的Triggers不需要写任何代码, 便可以打出对应的包。

上面所说的plist文件大体结构是这样的:
![](http://p28r7eh75.bkt.clouddn.com/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/exportPlist.png)

这个plist文件对应一系列的参数, 并不需要我们自己写, 手动打一次包, 即可导出这个文件。这里顺便提一句, Server配置好后, 连接此Server后, 任意机器都可以上传此plist文件, Server是将上传的plist文件拷贝到当前Bot工作目录下。

在Server配置好后, 即使是windows电脑也可以通过ip或者自签证书登录, 
https://192.168.0.xxx/xcode/lastest 或 https://xxxdemac-mini.local/xcode/bots/latest, 登陆后会显示以下界面, 点击integration, 便可以开始集成了, 但是这里似乎只能够集成, 不能配置, 不过根据Apple的惯性, 想要构造自己的生态, 我们也是能理解的。
![](http://p28r7eh75.bkt.clouddn.com/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/xcode_page.png)


## 关于jenkins的一些配置注意事项:
### 以下是我在配置过程中踩到的一些坑:

1. 8080端口被其他程序占用, 启动失败: java -jar jenkins.war —httpPort=8082;
2. git权限需要告诉jenkins私钥, 而不是git上的公钥: cat ~/.ssh/id_rsa;

![](http://p28r7eh75.bkt.clouddn.com/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/jenkins_rsa.png)

接下来, 其他用户直接通过浏览器登录 http://192.168.0.xxx:8082 , 通过账号密码登录, 便可以配置和构建项目。

### jenkins相对Mac OS Server的优点:

1. 同一局域网便可以登录, 登录之后便可以自行配置项目
2. 似乎可以并行构建任务

当使用Mac OS Server进行打包, 无论进行多少个打包任务, 它只开启一个xcodebuild进程
![](http://p28r7eh75.bkt.clouddn.com/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/xcodebuild_process_server.png)
而使用jenkins进行多项目打包, 这里开始构建两个项目就开启两个进程(下图上面两个xcodebuild进程是jenkins开启)
![](http://p28r7eh75.bkt.clouddn.com/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/xcodebuild_process_jenkins.png)
这里我没有做定量的测试, 猜想是jenkins的效率稍优, 对于多核处理器, 相同的计算能力, 对于两个构建来说, 应该没多大差距, 但对于拉代码等耗时操作, 比起Server其他构建任务在排队, 这部分就能省上一些时间。

但是jenkins有更方便的打包方式:
jenkins开启token, 不需要用户名登录便可以打包:

![](http://p28r7eh75.bkt.clouddn.com/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/jenkins_token.png)

这样给构建项目设置后还是不行的, 因为jenkins觉得这样是不安全的, 拿到了token就可以做任何事了。
系统管理->全局安全配置->勾选 Allow anonymous read access
![](http://p28r7eh75.bkt.clouddn.com/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/jenkins_allow_anonymous_read_access.png)

接着, 我们便可以通过命令来打包:
```
curl http://10.24.113.24:8082/job/notification_extension_test/build\?token\=123\&cause\=testBuild
```

| 参数       						| 注释    |
| --------   						| -----:   |
| notification_extension_test | 项目名称     |
| token       					| 上面设置的token    |
| cause        					| 可选参数, 可不传      |

这样似乎可以用一台服务器, 将打包任务部署到指定机器上:
![](http://p28r7eh75.bkt.clouddn.com/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/jenkins_servers.png)
这样可以在一台机器上集成公司不同端的项目, 而且还不影响打包效率。

## 关于Server和jenkins的一些总结:
1. 如果仅仅是iOS端的打包, Server是完全够用了, 而且操作贴近我们平时的开发风格, 虽然网页无法配置, 但是对于大部分公司来说, 打包配置都是开发在做的, 而不是测试;
2. 对于iOS端小型项目来说, 没有特别多的分支, 直接可以多建几个bot, 从而避开手写脚本;
3. 如果多端同一服务器, 那么jenkins无疑有较大的优势；如果公司有足够的电脑作为分布打包服务器, 那么打包效率会更上一层楼。

## fastlane及打包脚本简单介绍
说到自动化打包, 就不得不谈当下非常流行的fastlane, 如果说Server和jenkins是同一维度的, 都是打包平台, 那么fastlane应该是和shell脚本来作比较, 或者可以说, fastlane是在shell的基础上封装了一层, fastlane相比于脚本打包, 短暂体验后, 我觉得优点主要有:

1. 避免繁琐的路径拼接, 拷贝等
2. 修改工程配置文件, 避免调试时修改配置文件不小心提交到远程分支, 导致打包失败

我们来简单看一段fastlane的打包代码:
![](http://p28r7eh75.bkt.clouddn.com/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/fastlane_demo.png)

上述代码参数基本见名知意, 不难看出, 这基本就是给之前Server的exportPlist文件的一种包装, 只需执行

```
fastlane adhocMyApp version:100000  // 100000是传的版本号
```
便可以自动打出一个包, 并导出dSYM文件, 这里我故意把Distribution的provisioning Profile改成企业的
![](http://p28r7eh75.bkt.clouddn.com/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/fastlane_configuration.png)

发现工程配置文件发生了改变, 这里比较暴力, 把每种configuration的Provisioning Profile和teamID全都改了
![](http://p28r7eh75.bkt.clouddn.com/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/fastlane_change_configuration.png)

我们再看终端, 看看fastlane究竟做了些啥
![](http://p28r7eh75.bkt.clouddn.com/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/fastlane_temernal.png)

也确实和上图一样, 把所有都改成了AdHoc的。在进行修改配置后, 最终也是执行打包的核心脚本:

```
// 对应手动打包archive
xcodebuild archive -workspace ${work_space} -scheme ${scheme} -configuration ${configurationRelease} -archivePath ${archivePath}
// 对应导出步骤
xcodebuild -exportArchive -archivePath ${archivePath} -exportPath ${exportPath} -exportOptionsPlist ${exportOptionsPlist}

```

上述脚本的参数也基本见名知意, 脚本中${work_space}等代表取一个变量的值, 这里都是各个配置对应的路径或字符串。

经历上述脚本后, 就会在指定的exportPath路径下生成.ipa文件。我们一般是要将ipa和dSYM文件copy到指定的文件夹供测试去取, 后面便是一段处理繁琐的路径的脚本, 脚本本身没任何难度, 但是要格外注意, 且测试起来需要花费一定的时间, 如果使用fastlane就可以避免这个烦恼。

## 总结
本文主要是团队中的一次分享后的整理, 并不是特别细致的教程, 只是对当下的自动化打包的一些尝试及过程中遇到的一些问题和自己的一点思考, 如果有说的不对的地方, 望不吝赐教。