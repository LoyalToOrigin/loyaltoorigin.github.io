---
title: 关于iOS自动化打包的一些分享
date: 2018-01-11
categories: 
---

说到自动化打包，相信大家在日常开发中都有所接触，尤其是在多分支并行开发的情况下，自动化打包显得尤为重要，很多时候，我们打包一般是打及成分支的包，开发却在开发分支上，如果采取手动打包，我们需要反复切分支，不仅影响工作效率，而且会打断我们的开发思维，而却在工程较大的情况下，xcode每次indexing需要的时间就很久。

即使对于很多单分支开发的小项目来说，自动化打 包的优势也是不言而喻的，因为在手动打包的同时，基本可以说是什么事都做不了的，你需要一步步等待archive，export这些机械化的步骤。而有了自动化打包，你只需要点击一个按钮，便可以继续自己的开发。所以，自动化打包势在必行。

本文主要记录了我在公司自动化打包布置中的一些探索，及各平台的优缺点和配置过程踩过的坑。

谈到iOS的持续集成，我们首先想到的一定会是jenkins，这里我先介绍下我司采用的Mac OS Server(以下简称Server)这个平台的一些优缺点。

## Server相比于jenkins，我总结优点有三: 

1. 相比于jenkins的各种繁琐配置，Server配置简单，全程基本下一步操作即可；
2. 直接使用xcode就可开始构建项目，而不需要登录网页；
3. 集成度相当高，没有特别的需求，基本可以不写脚本，只需要配置一个plist文件即可以打包。
 
这里不做过多的配置介绍，虽然Server没有jenkins热门，但网上的文章也比比皆是，而且如上优点1中所说，Server配置真的很简单，在证书、描述文件齐全的情况下，基本就是一直点下一步操作。

下面我介绍使用过程中需要注意的一些方面: 

![xcode_integration.png](https://blog-1258097834.cos.ap-shanghai.myqcloud.com/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/xcode_integration.png?q-sign-algorithm=sha1&q-ak=AKIDYZ90MMp03Ltry7Q6lrVUesSF1Zy6Cnjn&q-sign-time=1542608545;1542610345&q-key-time=1542608545;1542610345&q-header-list=&q-url-param-list=&q-signature=09c48a51bb79b09d1369b80951eb64f15ac4dc8c&x-cos-security-token=6d896c8def4d043f8859667709972b086927e3e710001)

如上图所示，上图是对bot的各种设置，一个bot对应一个独立工作空间，如果有了解jenkins的话，bot可以类比jenkins的一个项目。

如果对打包没有特别需求，勾选Archive，选择对应Scheme、Configuration，指定一个plist文件，后面的Triggers不需要写任何代码，便可以打出对应的包。

上面所说的plist文件大体结构是这样的:
![exportPlist.png](https://blog-1258097834.cos.ap-shanghai.myqcloud.com/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/exportPlist.png?q-sign-algorithm=sha1&q-ak=AKIDAP0NLloSVKhZ6msGTrp5js3kv5R2Mvpz&q-sign-time=1542608593;1542610393&q-key-time=1542608593;1542610393&q-header-list=&q-url-param-list=&q-signature=d277106991a172c9692992b93b06d1bab00b0915&x-cos-security-token=611b3081032efb4191690e512d21ea9db20b6fd910001)

这个plist文件对应一系列的参数，并不需要我们自己写，手动打一次包，即可导出这个文件。这里顺便提一句，Server配置好后，连接此Server后，任意机器都可以上传此plist文件，Server是将上传的plist文件拷贝到当前Bot工作目录下。

在Server配置好后，即使是windows电脑也可以通过ip或者自签证书登录，
https://192.168.0.xxx/xcode/lastest 或 https://xxxdemac-mini.local/xcode/bots/latest，登陆后会显示以下界面，点击integration，便可以开始集成了，但是这里似乎只能够集成，不能配置，不过根据Apple的惯性，想要构造自己的生态，我们也是能理解的。
![xcode_page.png](https://blog-1258097834.cos.ap-shanghai.myqcloud.com/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/xcode_page.png?q-sign-algorithm=sha1&q-ak=AKIDbx0dRNzR0GuohR4P0t6yFJetGXVouFzY&q-sign-time=1542608659;1542610459&q-key-time=1542608659;1542610459&q-header-list=&q-url-param-list=&q-signature=6915f54f3a54c612163e27800799ae56d218bf68&x-cos-security-token=a670f434ef030dd36703936c82cd411dd8f3696f10001)


## 关于jenkins的一些配置注意事项:
### 以下是我在配置过程中踩到的一些坑:

1. 8080端口被其他程序占用，启动失败: java -jar jenkins.war —httpPort=8082；
2. git权限需要告诉jenkins私钥，而不是git上的公钥: cat ~/.ssh/id_rsa；

![jenkins_rsa.png](https://blog-1258097834.cos.ap-shanghai.myqcloud.com/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/jenkins_rsa.png?q-sign-algorithm=sha1&q-ak=AKIDq5FlNhAQUB6z06ihMrsbZ5ZbI6Vlzgin&q-sign-time=1542608794;1542610594&q-key-time=1542608794;1542610594&q-header-list=&q-url-param-list=&q-signature=26944b15d415f7bcc718ab3bc0161c5c8fd32ce7&x-cos-security-token=ab57e16b745ec6407df4953ba92253d2bfb82bb110001)

接下来，其他用户直接通过浏览器登录 http://192.168.0.xxx:8082 ，通过账号密码登录，便可以配置和构建项目。

### jenkins相对Mac OS Server的优点:

1. 同一局域网便可以登录，登录之后便可以自行配置项目
2. 似乎可以并行构建任务

当使用Mac OS Server进行打包，无论进行多少个打包任务，它只开启一个xcodebuild进程
![xcodebuild_process_server.png](https://blog-1258097834.cos.ap-shanghai.myqcloud.com/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/xcodebuild_process_server.png?q-sign-algorithm=sha1&q-ak=AKIDVc5k6H1o82yqNUEr4KFeCCDOfRT7SHK4&q-sign-time=1542608696;1542610496&q-key-time=1542608696;1542610496&q-header-list=&q-url-param-list=&q-signature=8a8dd0b3ba4f49a237faf013bb5d919bb8ce525f&x-cos-security-token=30ab79a4cdcc7c53d1df8615b2f724d9f6b5b54710001)
而使用jenkins进行多项目打包，这里开始构建两个项目就开启两个进程(下图上面两个xcodebuild进程是jenkins开启)
![xcodebuild_process_jenkins.png](https://blog-1258097834.cos.ap-shanghai.myqcloud.com/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/xcodebuild_process_jenkins.png?q-sign-algorithm=sha1&q-ak=AKIDR8RhpGgcd7mT7uVWuMSl31DW20u1G77L&q-sign-time=1542608839;1542610639&q-key-time=1542608839;1542610639&q-header-list=&q-url-param-list=&q-signature=0f16cddf60e6998fe72f23182e2203b3577c3ccc&x-cos-security-token=993f269a869e90900edcdc748c831d1a28f0f97610001)
这里我没有做定量的测试，猜想是jenkins的效率稍优，对于多核处理器，相同的计算能力，对于两个构建来说，应该没多大差距，但对于拉代码等耗时操作，比起Server其他构建任务在排队，这部分就能省上一些时间。

但是jenkins有更方便的打包方式:
jenkins开启token，不需要用户名登录便可以打包:

![jenkins_token.png](https://blog-1258097834.cos.ap-shanghai.myqcloud.com/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/jenkins_token.png?q-sign-algorithm=sha1&q-ak=AKIDGi4IHsjjBZxYgqGDHR1VIS4d6wbD8y31&q-sign-time=1542608884;1542610684&q-key-time=1542608884;1542610684&q-header-list=&q-url-param-list=&q-signature=3c50ce9fb2782b32e7a53d403d87ec887d956477&x-cos-security-token=2c55359d44b5eacc96d24568b3b5be3dccda548310001)

这样给构建项目设置后还是不行的，因为jenkins觉得这样是不安全的，拿到了token就可以做任何事了。
系统管理->全局安全配置->勾选 Allow anonymous read access
![jenkins_allow_anonymous_read_access.png](https://blog-1258097834.cos.ap-shanghai.myqcloud.com/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/jenkins_allow_anonymous_read_access.png?q-sign-algorithm=sha1&q-ak=AKIDS9HCUpyyNJ37EBC5FGs53N1TgiTti3QW&q-sign-time=1542608909;1542610709&q-key-time=1542608909;1542610709&q-header-list=&q-url-param-list=&q-signature=761db163edcbc74386f6d25fc198b033ff7a90a6&x-cos-security-token=c30c8c9ba3522a798067e38ff3e29e307235ecf810001)

接着，我们便可以通过命令来打包:
```
curl http://10.24.113.24:8082/job/notification_extension_test/build\?token\=123\&cause\=testBuild
```

| 参数       						| 注释    |
| --------   						| -----:   |
| notification_extension_test | 项目名称     |
| token       					| 上面设置的token    |
| cause        					| 可选参数，可不传      |

这样似乎可以用一台服务器，将打包任务部署到指定机器上:
![jenkins_servers.png](https://blog-1258097834.cos.ap-shanghai.myqcloud.com/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/jenkins_servers.png?q-sign-algorithm=sha1&q-ak=AKID3BbJPUm4jXxg12jjbdvMDAQGtvg5ctHO&q-sign-time=1542608941;1542610741&q-key-time=1542608941;1542610741&q-header-list=&q-url-param-list=&q-signature=4487425b6de1f861662fef55ab42a46b3bbd23de&x-cos-security-token=bca87019e36c4fba4cf5a0830305f9ed64ffffaa10001)
这样可以在一台机器上集成公司不同端的项目，而且还不影响打包效率。

## 关于Server和jenkins的一些总结:
1. 如果仅仅是iOS端的打包，Server是完全够用了，而且操作贴近我们平时的开发风格，虽然网页无法配置，但是对于大部分公司来说，打包配置都是开发在做的，而不是测试；
2. 对于iOS端小型项目来说，没有特别多的分支，直接可以多建几个bot，从而避开手写脚本；
3. 如果多端同一服务器，那么jenkins无疑有较大的优势；如果公司有足够的电脑作为分布打包服务器，那么打包效率会更上一层楼。

## fastlane及打包脚本简单介绍
说到自动化打包，就不得不谈当下非常流行的fastlane，如果说Server和jenkins是同一维度的，都是打包平台，那么fastlane应该是和shell脚本来作比较，或者可以说，fastlane是在shell的基础上封装了一层，fastlane相比于脚本打包，短暂体验后，我觉得优点主要有:

1. 避免繁琐的路径拼接，拷贝等
2. 修改工程配置文件，避免调试时修改配置文件不小心提交到远程分支，导致打包失败

我们来简单看一段fastlane的打包代码:
![fastlane_demo.png](https://blog-1258097834.cos.ap-shanghai.myqcloud.com/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/fastlane_demo.png?q-sign-algorithm=sha1&q-ak=AKIDyNWkrRfn8MUwLupnNPddb25LC2nXYYbz&q-sign-time=1542609004;1542610804&q-key-time=1542609004;1542610804&q-header-list=&q-url-param-list=&q-signature=821d7ab7ead4d5f17abd425a60448979c2ce738b&x-cos-security-token=1f21aedfc55c154f5c9cf8e435d76ea7545621db10001)

上述代码参数基本见名知意，不难看出，这基本就是给之前Server的exportPlist文件的一种包装，只需执行

```
fastlane adhocMyApp version:100000  // 100000是传的版本号
```
便可以自动打出一个包，并导出dSYM文件，这里我故意把Distribution的provisioning Profile改成企业的
![fastlane_configuration.png](https://blog-1258097834.cos.ap-shanghai.myqcloud.com/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/fastlane_configuration.png?q-sign-algorithm=sha1&q-ak=AKIDxnIsuEk459NZ9aNE70hpC9jy3vlAIwfY&q-sign-time=1542609028;1542610828&q-key-time=1542609028;1542610828&q-header-list=&q-url-param-list=&q-signature=a0c74846399f8ce1dc38ddaf74a9186bb0c757dc&x-cos-security-token=9ab11356802a7ce309506d1ccda69ec9adcf69f710001)

发现工程配置文件发生了改变，这里比较暴力，把每种configuration的Provisioning Profile和teamID全都改了
![fastlane_change_configuration.png](https://blog-1258097834.cos.ap-shanghai.myqcloud.com/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/fastlane_change_configuration.png?q-sign-algorithm=sha1&q-ak=AKIDSMoLYTN5inQKXYQJzXuOM7ZqI4BVYGik&q-sign-time=1542609062;1542610862&q-key-time=1542609062;1542610862&q-header-list=&q-url-param-list=&q-signature=e4f08bceed695abcd9ea3f08c4e866c6dfa42ceb&x-cos-security-token=915223d4dfd2bf34d1633bf950835a52052c032210001)

我们再看终端，看看fastlane究竟做了些啥
![fastlane_temernal.png](https://blog-1258097834.cos.ap-shanghai.myqcloud.com/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/fastlane_temernal.png?q-sign-algorithm=sha1&q-ak=AKIDk60b88yWt6abT6XP46jBY9lSnbqQRJeL&q-sign-time=1542609098;1542610898&q-key-time=1542609098;1542610898&q-header-list=&q-url-param-list=&q-signature=a392e72b67a624049a8e226f5c6ee6f917a2654b&x-cos-security-token=a5d87c9bda442495b2ab6284a7b9d7043ca7622510001)

也确实和上图一样，把所有都改成了AdHoc的。在进行修改配置后，最终也是执行打包的核心脚本:

```
// 对应手动打包archive
xcodebuild archive -workspace ${work_space} -scheme ${scheme} -configuration ${configurationRelease} -archivePath ${archivePath}
// 对应导出步骤
xcodebuild -exportArchive -archivePath ${archivePath} -exportPath ${exportPath} -exportOptionsPlist ${exportOptionsPlist}

```

上述脚本的参数也基本见名知意，脚本中${work_space}等代表取一个变量的值，这里都是各个配置对应的路径或字符串。

经历上述脚本后，就会在指定的exportPath路径下生成.ipa文件。我们一般是要将ipa和dSYM文件copy到指定的文件夹供测试去取，后面便是一段处理繁琐的路径的脚本，脚本本身没任何难度，但是要格外注意，且测试起来需要花费一定的时间，如果使用fastlane就可以避免这个烦恼。

## 总结
本文主要是团队中的一次分享后的整理，并不是特别细致的教程，只是对当下的自动化打包的一些尝试及过程中遇到的一些问题和自己的一点思考，如果有说的不对的地方，望不吝赐教。
