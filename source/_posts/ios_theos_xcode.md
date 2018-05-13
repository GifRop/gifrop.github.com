title: IOS越狱开发环境搭建
date: 2014-12-03 12:37:10
tags: [IOS越狱开发]
---
环境为Xcode6.1，sdk ios8.1<!-- more -->
{% codeblock lang:cpp %}
#安装最新的XCode
#安装Command Line Tools for XCode
打开XCode: File -> New -> Project -> Application ->Command Line Tool.
or使用命令行:sudo xcode-select --install

# clone theos.git 下载theos
cd ~
git clone git://github.com/DHowett/theos.git

# clone iphoneheaders.git 下载ios头文件,头文件放在theos/include目录下
cd ~/theos/
mv include include.bak
git clone git://github.com/rpetrich/iphoneheaders.git include
for FILE in include.bak/ *.h; do mv $FILE include/; done
rmdir include.bak/

# get IOSurfaceAPI.h 下载IOSurfaceAPI.h头文件，这个文件在上面的头文件里没有包含
cd ~/theos/include/IOSurface/
curl -O -k http://keykernel.org/other/IOSurfaceAPI.h

# clone theos-nic-templates.git 下载theos模板，默认只有5个，这里有5个
cd ~/theos/templates/
git clone git://github.com/orikad/theos-nic-templates.git

# get ldid for Mac OS X 下载ldid(必须的工具)
cd ~/theos/bin
curl -O http://keykernel.org/other/ldid
chmod a+x ldid

# get libsubstrate.dylib substrate.h 下载最新的hook框架和头文件
cd ~/theos
curl -OL http://apt.saurik.com/debs/mobilesubstrate_0.9.5101_iphoneos-arm.deb
dpkg-deb -x mobilesubstrate_0.9.5101_iphoneos-arm.deb mobilesubstrate
cp mobilesubstrate/Library/Frameworks/CydiaSubstrate.framework/CydiaSubstrate  ~/theos/lib/libsubstrate.dylib
cp mobilesubstrate/Library/Frameworks/CydiaSubstrate.framework/Headers/CydiaSubstrate.h include/substrate.h

#下载最新的MacPorts并安装，安装完重启CMD
https://www.macports.org/
#更新MacPorts,时间比较长，需要耐心等待，如果提示xcode授权有问题请在XCode里
#点击同意协议即可，或者在命令行里sudo xcodebuild -license，
#按任意键，最后Agree即可。
sudo port -v selfupdate

#下载dpkg用于deb打包
sudo port install dpkg

#到目前为止，基本上的环境都搭建好了，有些朋友还会使用IOSOpenDev
#可以到官网下载最新的版本
#安装失败可以换上代理试试，该工具会联网下载安装文件，某些网络
#无法下载，换个网络环境或者上代理即可。

#使用theos测试
$/opt/theos/bin/nic.pl
NIC 2.0 - New Instance Creator
------------------------------
  [1.] iphone/application
  [2.] iphone/cydget
  [3.] iphone/framework
  [4.] iphone/library
  [5.] iphone/notification_center_widget
  [6.] iphone/preference_bundle
  [7.] iphone/sbsettingstoggle
  [8.] iphone/tool
  [9.] iphone/tweak
  [10.] iphone/xpc_service
Choose a Template (required): 
#选择模板，接着包名，作者之类的可以自己设置好

#写好代码之后
make                //编译
make package        //编译成deb包
make package install//编译成deb包并上传到手机安装

#遇到一些导入的类无法链接的，把Makefile里的XXXX改为建立工程的包名，并添加对应的库
XXXXTweak_FRAMEWORKS = UIKit  //添加UIKit

#makefile的一些设置
export ARCHS = armv7 armv7s arm64  //编译生成文件的指令集，这里为三种都生成
export THEOS_DEVICE_IP=192.168.1.11 //设备的IP地址
export SDKVERSION=8.1               //IOS SDK版本

{% endcodeblock %}

