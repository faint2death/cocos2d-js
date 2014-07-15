# JSB调试

------

## 1. 编译时增加远程调试功能

------

修改项目目录下frameworks/runtime-src/Classes/AppDelegate.cpp，在sc->start();之后添加如下代码：

```
    sc->start();

#if defined(COCOS2D_DEBUG) && (COCOS2D_DEBUG > 0)
    sc->enableDebugger();   // Enable debugger here
#endif	
```

cocos2d-js使用firefox的SpiderMonkey作为javascript的解析引擎，新版本的SpiderMonkey支持远程的javascript调试功能。

## 2. firefox客户端配置调试

------

firefox的版本必须24以上，然后在地址栏输入`about:config`，找到`devtools.debugger.remote-enabled`项，双击，设置成`true`。其他的下面这个官方文档写的很详细了，不再重复：

https://github.com/cocos2d/cocos2d-js/tree/master/frameworks/js-bindings/bindings/script/debugger

还有一个问题，如果和官方文档一样调试模拟器，那么Host地址写`127.0.0.1`就行，但如果要调试真实的手机，那么在Connect to remote device的Host里，填手机wifi获取的IP即可。端口Port都是`5086`。


