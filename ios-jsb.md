# cocos2d-js 3.0 jsb环境调用底层Objective-C代码

------

## 1. 注册jsb函数

------

比android要简单，因为Xcode可以C++和Objective-C混编的。环境升级到cocos2d-js 3.0 rc0，准备给javascript加一个osInfo的函数，来判断用户的系统信息以及网络信息。

首先在项目目录下的frameworks/runtime-src/Classes/目录添加jsb_ios_info.h的头文件，内容如下：

```c
#include "cocos2d_specifics.hpp"
#include "cocos2d.h"

#import "sys/sysctl.h"

std::string os_info();
bool jsb_os_info(JSContext *cx, uint32_t argc, JS::Value *vp);
void register_jsb_os_info(JSContext* cx, JSObject* obj);
```

增加jsb_ios_info.mm文件，内容如下：

```c
#include "jsb_ios_info.h"

std::string os_info() {
    size_t size;
    int nR = sysctlbyname("hw.machine", NULL, &size, NULL, 0);
    char *machine = (char *)malloc(size);
    nR = sysctlbyname("hw.machine", machine, &size, NULL, 0);
    NSString *platform = [NSString stringWithCString:machine encoding:NSUTF8StringEncoding];
    free(machine);
     
    if ([platform isEqualToString:@"iPhone1,1"]) {
        platform = @"iPhone";
    } else if ([platform isEqualToString:@"iPhone1,2"]) {
        platform = @"iPhone 3G";
    } else if ([platform isEqualToString:@"iPhone2,1"]) {
        platform = @"iPhone 3GS";
    } else if ([platform isEqualToString:@"iPhone3,1"]||[platform isEqualToString:@"iPhone3,2"]||[platform isEqualToString:@"iPhone3,3"]) {
        platform = @"iPhone 4";
    } else if ([platform isEqualToString:@"iPhone4,1"]) {
        platform = @"iPhone 4S";
    } else if ([platform isEqualToString:@"iPhone5,1"]||[platform isEqualToString:@"iPhone5,2"]) {
        platform = @"iPhone 5";
    }else if ([platform isEqualToString:@"iPhone5,3"]||[platform isEqualToString:@"iPhone5,4"]) {
        platform = @"iPhone 5C";
    }else if ([platform isEqualToString:@"iPhone6,2"]||[platform isEqualToString:@"iPhone6,1"]) {
        platform = @"iPhone 5S";
    }else if ([platform isEqualToString:@"iPod4,1"]) {
        platform = @"iPod touch 4";
    }else if ([platform isEqualToString:@"iPod5,1"]) {
        platform = @"iPod touch 5";
    }else if ([platform isEqualToString:@"iPod3,1"]) {
        platform = @"iPod touch 3";
    }else if ([platform isEqualToString:@"iPod2,1"]) {
        platform = @"iPod touch 2";
    }else if ([platform isEqualToString:@"iPod1,1"]) {
        platform = @"iPod touch";
    } else if ([platform isEqualToString:@"iPad3,2"]||[platform isEqualToString:@"iPad3,1"]) {
        platform = @"iPad 3";
    } else if ([platform isEqualToString:@"iPad2,2"]||[platform isEqualToString:@"iPad2,1"]||[platform isEqualToString:@"iPad2,3"]||[platform isEqualToString:@"iPad2,4"]) {
        platform = @"iPad 2";
    }else if ([platform isEqualToString:@"iPad1,1"]) {
        platform = @"iPad 1";
    }else if ([platform isEqualToString:@"iPad2,5"]||[platform isEqualToString:@"iPad2,6"]||[platform isEqualToString:@"iPad2,7"]) {
        platform = @"ipad mini";
    } else if ([platform isEqualToString:@"iPad3,3"]||[platform isEqualToString:@"iPad3,4"]||[platform isEqualToString:@"iPad3,5"]||[platform isEqualToString:@"iPad3,6"]) {
        platform = @"ipad 3";
    }

    NSString *info = [NSString stringWithFormat:@"%@|%@", platform, [UIDevice currentDevice].systemVersion];

    std::string *str = new std::string([info UTF8String]);

    return *str;
}

bool jsb_os_info(JSContext *cx, uint32_t argc, JS::Value *vp) {
    jsval ret = std_string_to_jsval(cx, os_info());
    JS_SET_RVAL(cx, vp, ret);

    return true;
}

void register_jsb_os_info(JSContext *cx, JSObject *obj) {
    JS_DefineFunction(cx, obj, "osInfo", jsb_os_info, 0, 0);  
}
```

然后修改同目录下AppDelegate.cpp文件，添加相应的引用和注册javascript函数：

```c
...
#if (CC_TARGET_PLATFORM == CC_PLATFORM_IOS)
#include "jsb_ios_info.h"
#endif
...
    sc->addRegisterCallback(register_jsb_os_info);
    
#if (CC_TARGET_PLATFORM == CC_PLATFORM_ANDROID)
    sc->addRegisterCallback(JavascriptJavaBridge::_js_register);
#endif
    sc->start();
...
```

最后在Xcode的Classes目录添加jsb_ios_info.h和jsb_ios_info.mm这两个文件，编译后就可以在js端调用osInfo这个函数了。

## 2. javascript端执行效果

------

用javascript的osInfo函数返回大概是这样的：

```
iPhone 4S|7.1.2
```

整个过程比android简单的多。

## 3. 反射机制调用

------

在写android jsb文档的时候，提到ios有没有像android那样的反射机制调用，结果昨天看到cocos2d-js的github合并了一个新功能，就是反射机制调用Objective-C：

https://github.com/cocos2d/cocos2d-js/pull/574

有兴趣的童鞋可以试一下，但个人觉得意义不是很大，因为Xcode支持C++和Objective-C混编，反射机制在ios jsb情况下省不了什么的工作量。
