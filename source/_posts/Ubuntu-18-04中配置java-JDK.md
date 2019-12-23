---
title: Ubuntu 18.04中配置java JDK
date: 2019-11-23 23:38:03
tags:
- 开发环境搭建
categories:
- 杂项
---

#### 安装oracle jdk
下载所需的版本的JDK<https://www.oracle.com/technetwork/java/javase/downloads/jdk13-downloads-5672538.html>,然后解压:
```
sudo tar -zxvf jdk.....tar.gz
```
解压出来的目录就是oracle版本的jdk安装目录

#### 切换jdk版本
```
sudo update-alternatives --config java
```
<!-- more -->

可以看出在可配置列表中只有openjdk的选项, 没有oracle版本的jdk, 原因是Openjdk都是系统安装的,已经自动添加到可配置列表中,而oracle的jdk是手动解压到本地的,需要手动添加,添加方法:
```
sudo update-alternatives --install /usr/bin/java java /usr/lib/jvm/jdk安装文件名/bin/java 300
sudo update-alternatives --install /usr/bin/javac javac /usr/lib/jvm/jdk安装文件名/bin/javac 300
```

然后再执行
```
sudo update-alternatives --config java 
```
根据提示输入对应的编号,然后回车确认

#### 安装Android Studio
- 官网<https://developer.android.com/studio>下载Android Studio安装包
- 解压安装包
- 对android-studio进行配置
```
cd android-stuio/bin
./studio.sh
```
随后按照提示安装步骤进行安装
- 创建Desktop
安装的android-studio只能使用studio.sh进行启动不方便,建议创建desktop.

sudo vim /usr/share/applications/studio.desktop在其中编写如下代码:
```
[Desktop Entry]
Name=AndroidStudio
Comment=AndroidStudio
Exec=studio.sh路径
Icon=studio.png路径
Terminal=false
Type=Application
```