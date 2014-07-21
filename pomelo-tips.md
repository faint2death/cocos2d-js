# cocos2d-js 3.0使用pomelo若干问题

------

## 1. 客户端配置

------

### 1.1 WEB端

使用的是pomelo 0.9.10，但是pomelo生成项目里的build.js还是老版本，不支持reconnect，用下面连接的版本就可以：

https://github.com/NetEase/chatofpomelo-websocket/blob/master/web-server/public/js/lib/build/build.js

然后在project.json的jsList里直接引用即可：

```
    "jsList" : [
        "src/pomelo-cocos2d-jsb/build.js",
```

开发工具用的是Webstorm，平时逻辑功能的开发调试还是用WEB比较方便。

### 1.2 JSB环境

这个是开始在Android环境测试的结果。

在项目目录的frameworks/js-bindings/bindings/script目录下执行：

```bash
git clone https://github.com/Netease/pomelo-cocos2d-jsb.git --recursive
```

这样从pomelo官方下载pomelo在jsb环境下的客户端脚本，然后修改frameworks/js-bindings/bindings/script/jsb.js，在最后一行增加：

```bash
require('pomelo-cocos2d-jsb/index.js');
```

然后修改项目目录下frameworks/runtime-src/Classes/AppDelegate.cpp，在头文件声明最后添加：

```
#include "network/jsb_websocket.h"
```

然后在sc->start();之前添加如下代码，增加websocket的支持：

```
    sc->addRegisterCallback(register_jsb_websocket);
    sc->start();
```

另外在编译的时候，记得删除project.json的jsList里关于build.js的行，json文件里不能用注释，否则会`黑屏`。

### 1.3 IOS的JSB环境

编译IOS的ipa安装包时发现，frameworks/js-bindings/bindings/script/pomelo-cocos2d-jsb目录及以下文件都不会被打包到app里（3.0 beta），导致jsb.js运行的时候出错，手机`黑屏`。正好在[cocoachina论坛看到有位同学把pomelo-cocos2d-jsb放到src目录](http://www.cocoachina.com/bbs/read.php?tid=207220)，用project.json的jsList进行加载：

```
    "jsList" : [
        "src/pomelo-cocos2d-jsb/index.js",
```

还需要修改一下pomelo-cocos2d-jsb/index.js最后几个require的代码：

```
...
require('src/pomelo-cocos2d-jsb/lib/emitter/index.js');

window.EventEmitter = Emitter;

require('src/pomelo-cocos2d-jsb/lib/pomelo-protocol/lib/protocol.js');

require('src/pomelo-cocos2d-jsb/lib/pomelo-protobuf/lib/client/protobuf.js');

require('src/pomelo-cocos2d-jsb/lib/pomelo-jsclient-websocket/lib/pomelo-client.js');
```

这样android和ios就通用了，而且可以方便实现热更新。就是project.json文件针对WEB和JSB有不同，记得打包的时候改一下。

### 1.3 热更新环境

使用[热更新](assetsmanager.md)以后，project.json只加载`src/AssetsManager.js`文件，其它代码文件都通过它动态加载，所以它需要做一下判断是WEB环境还是JSB环境，从而加载不同的js文件：

```javascript
    loadGame:function(){
        cc.loader.loadJs(["src/files.js"], function(err){
            if (cc.sys.isNative) {
                // jsb环境要用jsb版本的pomelo
                jsFiles[0] = "src/pomelo-cocos2d-jsb/index.js";
            }
            cc.loader.loadJs(jsFiles, function(err){
                // 游戏入口
                Game.init();
            });
        });
    },
```

`src/files.js`文件列表第一个用WEB版本的pomelo文件，JSB替换这个文件。
