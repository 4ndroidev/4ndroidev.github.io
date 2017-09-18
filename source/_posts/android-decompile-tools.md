title: Android 反编译工具
categories: Android
tags:
  - Android
  - 反编译
date: 2017-04-17 12:01:46
---

## Apktool

> 将Apk反编译成资源文件及`smali`代码，将Jar反编译`smali`代码

- github: [https://github.com/iBotPeaches/Apktool](https://github.com/iBotPeaches/Apktool)

- website: [https://ibotpeaches.github.io/Apktool](https://ibotpeaches.github.io/Apktool)

- usage: [https://ibotpeaches.github.io/Apktool/documentation/](https://ibotpeaches.github.io/Apktool/documentation/)

- smali syntax: [http://www.netmite.com/android/mydroid/dalvik/docs/dalvik-bytecode.html](http://www.netmite.com/android/mydroid/dalvik/docs/dalvik-bytecode.html)

<!-- more -->

常用命令
  ```shell
  # 反编译
  apktool d xxx.apk

  # 回编译
  apktool b xxx

  # 参数
  # -r,--no-res             忽略资源文件
  # -s,--no-src             忽略代码文件
  ```

## dex2jar

> 将dex文件反编译成`*.class`集合的jar文件，之后可以使用`JD-GUI`工具查看

- github: [https://github.com/pxb1988/dex2jar](https://github.com/pxb1988/dex2jar)

常用命令: sh d2j-dex2jar.sh classes.dex

## JD-GUI

> 用于查看`*.class`集合的jar文件，如第三方sdk的jar包，或者dex2jar转换得到的jar文件

![image](/images/android-decompile-tools/jd-gui.jpg)

## IDA

> 用于查看so文件汇编指令，一般用于破解apk签名

## HexEdit

> 用于十六进制修改字符串或者部分指令，一般用于破解apk签名
