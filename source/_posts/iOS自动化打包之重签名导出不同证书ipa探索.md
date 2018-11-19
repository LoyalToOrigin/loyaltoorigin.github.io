---
title: iOS自动化打包之重签名导出不同证书ipa探索
date: 2018-03-08
categories: iOS
---

在完成基础的自动包打包流程过后，随即也出现了日常中常见的问题，比如我们每次需要打出不同网络环境和不同证书的ipa，由于开发者可以添加的设备只有100个，而公司的几个项目都是用的一个账号，各项目组都是独立的，再加上期间加入设备的员工的离职，真正能参与测试的设备寥寥无几。

所以我司一般测试都是使用企业证书，这样不同的项目都可以公用同一个证书，不仅管理起来方面，而且还摆脱了设备数量限制的烦恼，但另一方面，对于需要测试内购等功能的时候，仍然需要使用adhoc证书的包来进行测试。

我们原先的打包策略是通过执行脚本时输入的参数来打对应的包，这样对于不同测试并行测试，一次就要打出好几个，以我司作为打包服务器 Mac Mini 来说，archive + export 一个包的时间约为20min，对于不同证书不同环境的包随机组合，一次打出4个不同的包的时间就要花费约1h20min，而且在打包的时候，如果其他同事修改了新的bug，也无法打包。

因此，我们寻思能不能通过重签名的方式，只编译一次，对其重签名，打出不同的包。

本文主要介绍我在此过程中的一些探索，旨在提高不同证书不同环境的打包效率。


# 对ipa进行重签名

起初，我在网上查阅了相关资料，按照相关教程，却最终以失败告终。 如果有同学直接对ipa进行重签名成功的，希望不吝赐教。

我估摸着是不是内部做了什么验证，导致对ipa重签名无法成功。所以，我想可不可以不到ipa这步，更早地对其进行信息的修改以及重签名，权当一次尝试，即使失败也能在探索中学到新知识。最终，成功将原来打4个包需要1h20min的时间压缩到30min不到。
 

# 不等导出ipa，修改.xcarchive文件

.xcarchive文件是对项目进行手动archive，或执行以下脚本:

```
xcodebuild archive -workspace ${work_space} -scheme ${scheme} -configuration ${configurationDistribution} -archivePath ${archivePath}
```

如果对打包命令不是很了解的，可以查看我的上一篇文章文章:[关于iOS自动化打包的一些分享](https://loyaltoorigin.github.io/2018/01/11/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/)

首先，我们进入到 .xcarchive 文件目录，发现里面一个 Info.plist 文件，打开如下显示:
![xcarchive_infoplist.png](https://blog-1258097834.cos.ap-shanghai.myqcloud.com/iOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E4%B9%8B%E9%87%8D%E7%AD%BE%E5%90%8D%E5%AF%BC%E5%87%BA%E4%B8%8D%E5%90%8C%E8%AF%81%E4%B9%A6ipa%E6%8E%A2%E7%B4%A2/xcarchive_infoplist.png?q-sign-algorithm=sha1&q-ak=AKIDmHJcHISyxNVlImAQO2KKPM9hmR55QP6I&q-sign-time=1542607918;1542609718&q-key-time=1542607918;1542609718&q-header-list=&q-url-param-list=&q-signature=53ad257d6584c1eb42aa274605d2328d86250458&x-cos-security-token=0f4b29d2d934d689bb8ffaf070a81268b09755f810001)

我们可以看到里面有一些App必需的属性。


## 1. 修改 .xcarchive 的 Info.plist

此处，如果项目 Bundle Identifier 需要发生改变，则修改 CFBundleIdentifier 对应的值，并将 SigningIdentity 改成 Bundle Identifier 对应的证书，关于此处SigningIdentity的值，可在钥匙串中找到对应的证书，查看其信息，即为下图中(英文系统)的 Common Name 。

![certificate_info.png](https://blog-1258097834.cos.ap-shanghai.myqcloud.com/iOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E4%B9%8B%E9%87%8D%E7%AD%BE%E5%90%8D%E5%AF%BC%E5%87%BA%E4%B8%8D%E5%90%8C%E8%AF%81%E4%B9%A6ipa%E6%8E%A2%E7%B4%A2/certificate_info.png?q-sign-algorithm=sha1&q-ak=AKIDMWIy6EKtsLCaYBW667zWlTEiTwtXhSaF&q-sign-time=1542607858;1542609658&q-key-time=1542607858;1542609658&q-header-list=&q-url-param-list=&q-signature=5591f125235eeff1bed033104279f7498bf03557&x-cos-security-token=f6a95d171b721c4b12efa87f63da52e7c9deebf610001)


## 2. 修改 App Extension 相关信息

此步是对于项目 target 中如 notification extension 等从属 target，如果没有 App Extension ，直接可以跳过此步，查看下一步 **修改主target相关信息** 。

通过文件夹打开 YourAppName.xcarchive/Products/Applications/YourAppName.app/PlugIns/YourAppNameNotificationServiceExtension.appex ，这里不是标准文件夹，open 命令似乎不起作用，观察其目录结构:

![extension_floder.png](https://blog-1258097834.cos.ap-shanghai.myqcloud.com/iOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E4%B9%8B%E9%87%8D%E7%AD%BE%E5%90%8D%E5%AF%BC%E5%87%BA%E4%B8%8D%E5%90%8C%E8%AF%81%E4%B9%A6ipa%E6%8E%A2%E7%B4%A2/extension_floder.png?q-sign-algorithm=sha1&q-ak=AKIDaVkaOH5z1HTMBTy0zyocEz56sJ0nJLEi&q-sign-time=1542607951;1542609751&q-key-time=1542607951;1542609751&q-header-list=&q-url-param-list=&q-signature=ce6c299252f87d589282667d944a2a304553a9f8&x-cos-security-token=54f6f8562fe3237462c78a8a4228d0aa6b6aafc510001)


### 2.1 修改 Info.plist 相关信息

App Extension 的 Bundle Identifier 是 App 的 Bundle Identifier 加上其对应后缀，如 notificationserviceextension 。

修改 Bundle Identifier 为对应的值，这里对应的值是指之前修改 .xcarchive 目录中 Info.plist 的 Bundle Identifier 对应，如 **com.test.www** ，这里便是 **com.test.www.notificationserviceextension**。

![extension_infoplist.png](https://blog-1258097834.cos.ap-shanghai.myqcloud.com/iOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E4%B9%8B%E9%87%8D%E7%AD%BE%E5%90%8D%E5%AF%BC%E5%87%BA%E4%B8%8D%E5%90%8C%E8%AF%81%E4%B9%A6ipa%E6%8E%A2%E7%B4%A2/extension_infoplist.png?q-sign-algorithm=sha1&q-ak=AKIDsBixLGiHWpo81vpHAMYgT17s4ydlcjCM&q-sign-time=1542607986;1542609786&q-key-time=1542607986;1542609786&q-header-list=&q-url-param-list=&q-signature=cbf245d2c304ee9162f27f658d5076e5e169a8fd&x-cos-security-token=4f3dc9971b738a57f0909eb29f37208f53d798be10001)

### 2.2 替换 Provisioning Profile

将对应的 Provisioning Profile 拷贝到该目录下替换原来的 Provisioning Profile ，改成相同的文件名 **embedded.mobileprovision** 。


### 2.3 修改 archived-expanded-entitlements.xcent

我们通过xcode打开archived-expanded-entitlements.xcent，其本质就是plist文件，
格式是 **teamId.bundle identifier** 。

![extension_archived-expanded-entitlements.png](https://blog-1258097834.cos.ap-shanghai.myqcloud.com/iOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E4%B9%8B%E9%87%8D%E7%AD%BE%E5%90%8D%E5%AF%BC%E5%87%BA%E4%B8%8D%E5%90%8C%E8%AF%81%E4%B9%A6ipa%E6%8E%A2%E7%B4%A2/extension_archived-expanded-entitlements.png?q-sign-algorithm=sha1&q-ak=AKIDLDw6XKrG3XABY7b79aLKmkpJJnPVnG1d&q-sign-time=1542608018;1542609818&q-key-time=1542608018;1542609818&q-header-list=&q-url-param-list=&q-signature=d9049b9af7190becf6f1e345681cbe9aff63da85&x-cos-security-token=3eb8208c733f1046eac0502a08e31360b7e9940010001)

修改图中遮盖的两项值，依旧是要和.xcarchive的Info.plist值对应。

### 2.4 重签名

用对应的证书对 App Extension 重新签名，这里的 **YourCetificateName** 依旧是修改 .xcarchive的Info.plist 里的证书名。

```
codesign -f -s "YourCetificateName" YourAppNameNotificationServiceExtension.appex
```


## 3. 修改主target相关信息

与上一步修改 App Extension 步骤基本相同，只是少一步，不用修改 archived-expanded-entitlements.xcent 。

### 3.1 修改Info.plist的相关信息

进入.app目录，修改Info.plist的Bundle Identifier，使其与.xcarchive文件对应。

你也可以修改其他一些值，如网络环境，是测试环境，还是生产环境，这里只是抛砖引玉。事实上，修改网络环境有方便的方法，如通过读取粘贴板的文本来切换，或者写一个辅助程序来打开我们的App，从而通知切换环境。

### 3.2 替换Provisioning Profile

将对应的 Provisioning Profile 拷贝到该目录下替换原来的 Provisioning Profile ，改成相同的文件名 **embedded.mobileprovision** 。


### 3.3 重签名

用对应的证书对 .app文件 重新签名，这里的 **YourCetificateName** 依旧是修改 .xcarchive的Info.plist 里的证书名。

```
codesign -f -s "YourCetificateName" YourAppName.app
```

## 4. 导出包

```
xcodebuild -exportArchive -archivePath YourAppName.xcarchive -exportPath $(pwd) -exportOptionsPlist YourExportOptionsPlistPath
```
成功后，命令台输出:

![export_succeed.png](https://blog-1258097834.cos.ap-shanghai.myqcloud.com/iOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E4%B9%8B%E9%87%8D%E7%AD%BE%E5%90%8D%E5%AF%BC%E5%87%BA%E4%B8%8D%E5%90%8C%E8%AF%81%E4%B9%A6ipa%E6%8E%A2%E7%B4%A2/export_succeed.png?q-sign-algorithm=sha1&q-ak=AKIDrMk3YrR1zHTPBA6WNLCSlh3YGG7nncu0&q-sign-time=1542608080;1542609880&q-key-time=1542608080;1542609880&q-header-list=&q-url-param-list=&q-signature=d32acd97b75cb80f1a0dfbb8c5a21bd3d4b54e0e&x-cos-security-token=24c82afd92277e2733368b652d4b7822d0947ee510001)

如果对于 **exportOptionsPlist** 不了解的，也可以看我的上篇文章:[关于iOS自动化打包的一些分享](https://loyaltoorigin.github.io/2018/01/11/%E5%85%B3%E4%BA%8EiOS%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85%E7%9A%84%E4%B8%80%E4%BA%9B%E5%88%86%E4%BA%AB/)
。

# 注意点

上述修改的每一步，无论是Bundler Identifer，还是Provisioning Profile，还是重签名用的证书，都是需要相对应的，如果有一步错了，ipa包是导不出来的。

我的表述可能不是那么清楚，相信大家操作一次，一步一步来，修改需要修改的值，其实基本是一目了然的。
大家如果有类似需求，建议先操作一次，成功后再写脚本实现自动化。

# 总结

经过上述操作，实质上只进行了一次编译，然后修改相关信息，导出对应不同的证书的包，只是多做了几次导出操作，大大地节省了打包时间。大家如果有什么想法或更好的办法，欢迎一起讨论讨论。





