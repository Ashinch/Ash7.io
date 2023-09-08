---
title: 利用 KDE-Connect 互联你的 macOS 与 Android 设备
date: 2021-06-15
tags: 
- Android
- macOS
---

当你同时拥有一台 macOS 系统的电脑和一台 Android 系统的手机时，有时会让人感觉很苦恼，它们既不像 macOS + iOS 那样能够使用 iCloud 同步、接力、隔空投送，也无法像 Windows + Android 那样采用 Cortana、Samsung、Your Phone APP、Huawei Share、MIUI+ 等互联方案，而 MIUI+ 的 macOS 版本又遥遥无期，macOS + Android 的搭配就处于这种尴尬的境地中（当然也有 AirDroid 方案）。

## KDE-Connect

[KDE Connect](https://kdeconnect.kde.org/)

KDE 是一款 Linux 桌面环境，而 KDE-Connect 就是 KDE 官方出品的**局域网设备互联开源软件**，它还支持移植到其他的 Linux 桌面环境、Android 系统、Windows 系统甚至是 iOS 、macOS 系统上，所以我们可以试图利用 KDE-Connect 来摆脱这种「信息孤岛」的情况。

KDE-Connect 的 macOS 端需要自行构建，在 [binary-factory-kde](https://binary-factory.kde.org/) 上自动构建的有官方远古版本和一个比较新的 Nightly 版本，目前是由[诺奇诺奇](https://www.zhihu.com/people/inokinoki)一人维护，不过目前 Nightly 版本的 APP 不知道为什么在 Catalina 上无法启动，找了一圈资料和我自行编译后的结果也是如此，最后还是在 [omgubuntu](https://www.omgubuntu.co.uk/2019/07/kde-connect-macOS-os-integrate-android) 上找到了一个版本终于可以正常启动了。

🚨 **注意：在不知道他人修改源代码的情况下，请勿轻易安装非官方编译的程序！**

![KDE-Connect for macOS](https://bucket-ashinch.oss-cn-hangzhou.aliyuncs.com/uPic/截屏2021-06-15%2021.41.10.png)

KDE-Connect for macOS

KDE-Connect 在其原生平台上有很多丰富的插件来提供各种各样的功能，Android 端也可以在 [Play Store](https://play.google.com/store/apps/details?id=org.kde.kdeconnect_tp) 或者 [F-Droid](https://f-droid.org/packages/org.kde.kdeconnect_tp/) 上获取。

![KDE-Connect for Android](https://bucket-ashinch.oss-cn-hangzhou.aliyuncs.com/uPic/Screenshot_2021-06-15-22-49-28-686_org.kde.kdeconnect_tp.jpg)

KDE-Connect for Android

Android 版本一切运行良好，但在 macOS 上的体验就不是那么的尽如人意了：

- [x] 手机电量监控
- [x] 剪贴板同步（Android 10 以上通过刷入 Kr328 制作的 Magisk [剪贴板白名单模块](https://github.com/Kr328/Riru-ClipboardWhitelist-Magisk)解决后台应用访问剪贴板的问题，Android 11 亲测 v9 版本可以无感双向同步）
- [x] 文件共享
- [x] 通知同步（应用、短信、电话）
- [x] 手机作为触摸板和键盘（卡顿、丢失精度）
- [ ] 远程命令
- [ ] 电脑多媒体控制（异常）
- [ ] 电脑发送短信（异常）

![KDE-Connect for macOS 设备控制菜单](https://bucket-ashinch.oss-cn-hangzhou.aliyuncs.com/uPic/截屏2021-06-15-22.32.39.png)

KDE-Connect for macOS 设备控制菜单

![KDE-Connect for macOS 同步通知](https://bucket-ashinch.oss-cn-hangzhou.aliyuncs.com/uPic/QQ20210615-223616.png)

KDE-Connect for macOS 同步通知

![KDE-Connect for macOS 设备文件 SFTP 浏览器](https://bucket-ashinch.oss-cn-hangzhou.aliyuncs.com/uPic/截屏2021-06-15-22.40.03.png)

KDE-Connect for macOS 设备文件 SFTP 浏览器

尽管可以正常启动，但 macOS 平台的 KDE-Connect 依然是不稳定的，不定期会出现一些问题：

- 无声无息的丢失连接
- 莫名其妙的找不到设备
- 在 macOS 上点击同步过来的通知时服务会 down 掉
- 有时断连后需要手机亮屏唤醒后才能够发包重连
- 当设备断连后需要退出重新运行 KDE-Connect 才能重新连接

通过[诺奇诺奇的文章](https://zhuanlan.zhihu.com/p/157779710)得知了一些使用的小技巧，macOS 的无线网络不支持 1500 以上的 MTU，发不出 UDP 广播识别包，所以在配对设备时首先在 Android 端发包刷新设备，然后从 Android 端发送配对请求，最后由 macOS 端来确认配对，这样就可以保持一个长足的连接。

## Soduto

在焦灼于 KDE-Connect 的不稳定时，我找到了这款叫做 [Soduto](https://soduto.com/) 的开源软件，它也是一款基于 KDE-Connect 的 macOS 平台客户端（作者一开始在 [GitHub](https://github.com/soduto/Soduto) 上都没提及是其客户端，怪不得难找），相对于原来的 KDE-Connect，Soduto 拥有出色的稳定性，从我使用到现在从未无故断连，并且出趟门回来后，当最近配对过的 Android 设备重回局域网时，依然可以自动且快速地恢复连接。而且在进行文件共享时可以直接将文件拖到顶部菜单栏的 Soduto 图标上即可发送，非常方便。

![https://bucket-ashinch.oss-cn-hangzhou.aliyuncs.com/uPic/截屏2021-06-15-22.27.53.png](https://bucket-ashinch.oss-cn-hangzhou.aliyuncs.com/uPic/截屏2021-06-15-22.27.53.png)

但 Soduto 也有很多缺点，这是一个在 2018 年就已经停止维护的应用，默认情况下它只有这些功能：

- 剪贴板同步
- 手机电量监控（看不到具体百分比）
- 文件共享（文件夹没有对应图标，但是可以双击打开文件）

![Soduto 设备控制菜单](https://bucket-ashinch.oss-cn-hangzhou.aliyuncs.com/uPic/截屏2021-06-15-22.55.26.png)

Soduto 设备控制菜单

![Soduto 设备文件 SFTP 浏览器](https://bucket-ashinch.oss-cn-hangzhou.aliyuncs.com/uPic/截屏2021-06-15-23.02.53.png)

Soduto 设备文件 SFTP 浏览器

## 动手！

想要 Soduto 的稳定性，又想要 KDE-Connect 必要的功能，那么就要自己来动手了。值得庆幸的是 Soduto 是一款开源软件，并且是使用 Swift 编写的原生 macOS 应用，对 KDE-Connect 也做了较为完善的封装。

1. 按照给出的 [README.md](https://github.com/soduto/Soduto#readme) 上描述的步骤开始安装依赖，并将对应的源代码 Clone 下来。
2. 在 Xcode 中打开，并设置个人开发者账户信息。
3. 然后就可以在 Xcode 或者 AppCode 上开始 Hacking。

### 为 Soduto 开启 Android 通知同步服务

![Soduto/AppDelegate.swift 60:9](https://bucket-ashinch.oss-cn-hangzhou.aliyuncs.com/uPic/截屏2021-06-15-23.40.20.png)

Soduto/AppDelegate.swift 60:9

通过浏览源代码时发现作者其实已经实现了通知同步服务，只不过是将其注释掉了，打开这行注释 `self.serviceManager.add(service: NotificationsService())` 即可开启通知同步服务，但作者并没有给出 macOS 通知时，右侧显示来源应用图标的实现，以后有时间再来完善一下。

![Soduto 同步通知](https://bucket-ashinch.oss-cn-hangzhou.aliyuncs.com/uPic/QQ20210615-225718.png)

Soduto 同步通知

### 为 Soduto 显示 Android 设备具体电量百分比

Soduto 默认是不显示 Android 设备的百分比电量信息的，只会显示是否在充电，仅能靠一个大致的电量图标来分辨：

![https://bucket-ashinch.oss-cn-hangzhou.aliyuncs.com/uPic/截屏2021-06-15-23.28.34.png](https://bucket-ashinch.oss-cn-hangzhou.aliyuncs.com/uPic/截屏2021-06-15-23.28.34.png)

可以在对应的 Swift UI 控制器文件里更改刷新菜单列表时的行为，让其刷新菜单列表时顺便把电量百分比写到设备名的前面，因为下面有方法复用，所以稍微改得多了一点，重点在 152 ~ 155 行：

![Soduto/UI/Controllers/StatusBarMenuController.swift 152:13](https://bucket-ashinch.oss-cn-hangzhou.aliyuncs.com/uPic/截屏2021-06-15-23.37.36.png)

Soduto/UI/Controllers/StatusBarMenuController.swift 152:13

重新编译后看看效果：

![https://bucket-ashinch.oss-cn-hangzhou.aliyuncs.com/uPic/截屏2021-06-15-23.44.28.png](https://bucket-ashinch.oss-cn-hangzhou.aliyuncs.com/uPic/截屏2021-06-15-23.44.28.png)

### 为 Android 端 KDE-Connect APP 实现断连后发送通知

Android 端的 KDE-Connect APP 默认会常驻一个通知在通知栏中，并提供一个发送剪贴板和一个发送文件的按钮，因为 Android 10 之后会禁止应用在后台访问剪贴板，所以我通过刷入 Magisk 的剪贴板白名单模块解决了这个问题，我一般很少从 Android 端发送文件，必要时也可以通过分享给 KDE-Connect APP 自动发送，所以我会关闭这个常驻的状态通知。

我希望当我的 Android 设备与 macOS 设备失去连接时能够弹出一条通知告知我，并且在重回局域网连接后也这么做，所以我又对 KDE-Connect APP 的 Android 版本进行了修改，Android 端也是使用 Java 编写的原生应用，所以改起来会顺手许多，从官方获取源码后，浏览一遍大致架构，最后选择在 `org/kde/kdeconnect/BackgroundService.java` 后台服务文件中，在定期检查设备连接状态的回调方法里，实现设备连接或者断开后紧接着发送一条普通通知。

![org/kde/kdeconnect/BackgroundService.java 197:13](https://bucket-ashinch.oss-cn-hangzhou.aliyuncs.com/uPic/截屏2021-06-16-00.01.13.png)

org/kde/kdeconnect/BackgroundService.java 197:13

![org/kde/kdeconnect/BackgroundService.java 215:13](https://bucket-ashinch.oss-cn-hangzhou.aliyuncs.com/uPic/截屏2021-06-16-00.01.31.png)

org/kde/kdeconnect/BackgroundService.java 215:13

最后打个包到手机上看看效果：

![https://bucket-ashinch.oss-cn-hangzhou.aliyuncs.com/uPic/1623773316976.jpg](https://bucket-ashinch.oss-cn-hangzhou.aliyuncs.com/uPic/1623773316976.jpg)

## 写在最后

第三方互联方案最终还是无法达到官方互联方案的体验效果，但目前我们仍然可以通过这些手段来满足平台差异间的设备互联。

对于 KDE-Connect 和 Soduto 来说，它们的高度还不止于此，你也可以自己动手来实现更酷的 feature，比如说在 Android 上收到验证码短信时，识别验证码并发送给 macOS（已实现 👉 [GitHub](https://github.com/Ashinch/kdeconnect-android/releases/tag/v1.17.0-03c4fe)），不需要再掏出手机，直接在输入框上粘贴即可完成（类似 Apple 接力），这些都由你的想象力来决定它们的能力 👍 。
