## cocos2d-js android平台安装包编译打包

------

### 1. 环境配置

cocos2d-js 3.0 rc0的changelog提到cocos在使用release模式编译的时候，自动把js脚本编译成jsc，但是看`cocos2d-js-v3.0-rc0/tools/cocos2d-console/plugins/plugin_jscompile/bin/`下的jsbcc只支持mac和win32平台，所以把原来linux下编译环境换到mac平台。

MacBook Air的机器，操作系统版本是OS X 10.9.3，Xcode版本是5.1.1，java开发环境已经装好。下载如下必要的开发包：

- [cocos2d-js-v3.0-rc0.zip](http://www.cocos2d-x.org/download)
- [android-ndk-r9d-darwin-x86.tar.bz2](https://developer.android.com/tools/sdk/ndk/index.html)
- [adt-bundle-mac-x86_64-20140624.zip](https://developer.android.com/sdk/index.html)
- [apache-ant-1.9.4-bin.tar.bz2](http://ant.apache.org/bindownload.cgi)

有新的可以下载更新的版本。

配置cocos2d-js：

```bash
xtekiMacBook-Air:cocos2d-js-v3.0-rc0 x$ ./setup.py 

Setting up cocos2d-x...
->Check environment variable COCOS_CONSOLE_ROOT
  ->Find environment variable COCOS_CONSOLE_ROOT...
    ->COCOS_CONSOLE_ROOT is found : /Users/x/cocos2d/cocos2d-js-v3.0-rc0/tools/cocos2d-console/bin

->Configuration for Android platform only, you can also skip and manually edit "/Users/x/.bash_profile"

->Check environment variable NDK_ROOT
  ->Find environment variable NDK_ROOT...
    ->NDK_ROOT is found : /Users/x/cocos2d/android-ndk-r9d

->Check environment variable ANDROID_SDK_ROOT
  ->Find environment variable ANDROID_SDK_ROOT...
    ->ANDROID_SDK_ROOT is found : /Users/x/cocos2d/adt-bundle-mac-x86_64-20140624/sdk

->Check environment variable ANT_ROOT
  ->Find environment variable ANT_ROOT...
    ->ANT_ROOT is found : /Users/x/cocos2d/apache-ant-1.9.4/bin


Please execute command: "source /Users/x/.bash_profile" to make added system variables take effect

xtekiMacBook-Air:cocos2d-js-v3.0-rc0 x$ source ~/.bash_profile
```

进入android sdk的目录下载更新android平台和第三方库文件：

```bash
xtekiMacBook-Air:cocos2d-js-v3.0-rc0 x$ cd ~/cocos2d/adt-bundle-mac-x86_64-20140624/sdk/
xtekiMacBook-Air:sdk x$ tools/android
```

这时会弹出`Android SDK Manager`的窗口，勾选`Android 2.3.3(API 10)`，然后安装。安装完后，在sdk的platforms下出现一个`android-10`的目录。

### 2. 编译测试

生成一个测试项目：

```bash
xtekiMacBook-Air:cocos2d-js-v3.0-rc0 x$ cocos new MyGame -l js -d ~/project/
```

修改`frameworks/runtime-src/proj.android/AndroidManifest.xml`可以设置游戏启动是横屏还是竖屏，默认是`landscape`：

```
android:screenOrientation="portrait"
```

编译这个测试项目：

```bash
xtekiMacBook-Air:cocos2d-js-v3.0-rc0 x$ cd ~/project/MyGame/
xtekiMacBook-Air:MyGame x$ cocos compile -p android
Runing command: compile
Building mode: debug
building native
NDK build mode: debug
The Selected NDK toolchain version was 4.8 !
...
debug:

BUILD SUCCESSFUL
Total time: 14 seconds
Move apk to /Users/x/project/MyGame/runtime/android
build succeeded.
xtekiMacBook-Air:MyGame x$ ls -l /Users/x/project/MyGame/runtime/android
total 15416
-rw-r--r--  1 x  staff  7889244 Jul 10 15:55 MyGame-debug.apk
```

这样编译的是debug版本，方便测试调试。一般开发阶段，这样就可以了。

### 3. 发布release版本

如果要编译release版本，那么首先要创建keystore：

```bash
xtekiMacBook-Air:MyGame x$ keytool -genkey -alias demo.keystore -keyalg RSA -validity 3650 -keystore demo.keystore
```

- 说明：
	- genkey 产生密钥
	- -alias demo.keystore 别名 demo.keystore
	- -keyalg RSA 使用RSA算法对签名加密
	- -validity 3650 有效期限10年
	- -keystore demo.keystore

会要求输入keystore的密码，以及组织名字等相关信息，填好后在当前目录生成keystore文件：

```bash
xtekiMacBook-Air:MyGame x$ ls -l demo.keystore
-rw-r--r--  1 x  staff  1334 Jul 10 14:52 demo.keystore
```

然后用如下命令编译发布：

```bash
xtekiMacBook-Air:MyGame x$ cocos compile -p android -m release
```

编译中间可以看到js被转成jsc文件。编译完后，会提示你输入keystore文件：

```bash
BUILD SUCCESSFUL
Total time: 12 seconds
Move apk to /Users/x/project/MyGame/publish/android
Please input the absolute/relative path of ".keystore" file:
```

输入刚才生成的demo.keystore完整路径，比如这里是`/Users/x/project/MyGame/demo.keystore`，然后会提示输入这个keystore的密码，alias以及alias的密码，然后脚本会自动做签名，生成做了签名的apk文件：

```bash
xtekiMacBook-Air:MyGame x$ ls -l /Users/x/project/MyGame/publish/android
total 24504
-rw-r--r--  1 x  staff  6271887 Jul 10 16:01 MyGame-release-signed.apk
-rw-r--r--  1 x  staff  6268000 Jul 10 16:01 MyGame-release-unsigned.apk
```

release模式编译的apk包比debug模式要小不少，解开apk包，查看assets目录下的js文件都编译成jsc文件，不用担心源码泄漏了。
