# cocos2d-js 3.0 android环境调用底层java代码

------

## 1. C++端工作

------

环境还是cocos2d-js 3.0 beta，准备给javascript加一个osInfo的函数，来判断用户的系统信息以及网络信息。

首先在项目目录下的frameworks/runtime-src/Classes/目录添加jsb_os_info.hpp，内容如下：

```c
#include "cocos2d_specifics.hpp"  
#include "cocos2d.h"

#include <jni.h>
#include "platform/android/jni/JniHelper.h"

/**c++调用java中的方法*/
std::string call_java() {
    cocos2d::JniMethodInfo osInfoBoard;

    bool isHave = cocos2d::JniHelper::getStaticMethodInfo(
        osInfoBoard,
        "org/cocos2dx/javascript/AppActivity", // 改的话对应frameworks/runtime-src/proj.android/src下修改
        "osInfoBoardStatic",
        "()Ljava/lang/String;"
    );
    if (!isHave) {
        cocos2d::CCLog("jni:osInfoBoardStatic false");
        return NULL;
    }

    jstring jstr = (jstring)osInfoBoard.env->CallStaticObjectMethod(osInfoBoard.classID, osInfoBoard.methodID);
    return cocos2d::JniHelper::jstring2string(jstr);
}

/**定义用来处理js请求的函数：
*cx:JS的上下文
*argc:参数的个数
*vp：具体的参数列表
*/
bool jsb_os_info(JSContext *cx, uint32_t argc, JS::Value *vp) {
    jsval ret = std_string_to_jsval(cx, call_java());
    JS_SET_RVAL(cx, vp, ret);

    return true;
}

/**在AppDelegate.cpp中被注册的函数：sc->addRegisterCallback(register_jsb_os_info);*/
void register_jsb_os_info(JSContext* cx, JSObject* obj) {     
    //绑定：“osInfo”为开发给js调用的方法
    JS_DefineFunction(cx, obj, "osInfo", jsb_os_info, 0, 0);  
}

```

然后修改同目录下AppDelegate.cpp文件，添加相应的引用和注册javascript函数：

```c
...
#if (CC_TARGET_PLATFORM == CC_PLATFORM_ANDROID)
#include "jsb_os_info.hpp"
#include "platform/android/CCJavascriptJavaBridge.h"
#endif
...
#if (CC_TARGET_PLATFORM == CC_PLATFORM_ANDROID)
    sc->addRegisterCallback(register_jsb_os_info);
    sc->addRegisterCallback(JavascriptJavaBridge::_js_register);
#endif
    sc->start();
...
```

注意顺序，其他非相关代码都省略了。

## 2. java端工作

------

修改项目目录frameworks/runtime-src/proj.android/src/org/cocos2dx/javascript/AppActivity.java，这个程序本来是空的，增加代码如下：

```java
package org.cocos2dx.javascript;

import org.cocos2dx.lib.Cocos2dxActivity;
import android.content.Context;
import android.net.ConnectivityManager;
import android.net.NetworkInfo;
import android.net.wifi.WifiManager;
import android.net.wifi.WifiInfo;
import android.telephony.TelephonyManager;

public class AppActivity extends Cocos2dxActivity {
    private static Context mContext;

    private static String formatIp(int ip) {
        return  ( ip & 0xFF ) + "." +
                ((ip >> 8 ) & 0xFF) + "." +
                ((ip >> 16 ) & 0xFF) + "." +
                ((ip >> 24 ) & 0xFF) ;
    }

    public static NetworkInfo getNetworkInfo(Context context){
        ConnectivityManager cm = (ConnectivityManager) context.getSystemService(Context.CONNECTIVITY_SERVICE);
        return cm.getActiveNetworkInfo();
    }

    public static boolean isConnected(Context context){
        NetworkInfo info = AppActivity.getNetworkInfo(context);
        return (info != null && info.isConnected());
    }

    public static String getWifi(Context context){
        NetworkInfo info = AppActivity.getNetworkInfo(context);
        if (info != null && info.isConnected() && info.getType() == ConnectivityManager.TYPE_WIFI) {
            WifiManager wifiManager = (WifiManager) context.getSystemService(Context.WIFI_SERVICE);
            WifiInfo wifiInfo = wifiManager.getConnectionInfo();

            return "Wifi|" + WifiManager.calculateSignalLevel(wifiInfo.getRssi(), 10) + "|" + wifiInfo.getLinkSpeed();
        } else {
            return "";
        }
    }

    public static String getMobile(Context context){
        NetworkInfo info = AppActivity.getNetworkInfo(context);
        if (info != null && info.isConnected() && info.getType() == ConnectivityManager.TYPE_MOBILE) {
            String mobileType = AppActivity.getMobileType(info.getSubtype());

            return "Mobile|" + mobileType;
        } else {
            return "";
        }
    }

    public static String getMobileType(int subType){
        switch(subType){
            case TelephonyManager.NETWORK_TYPE_1xRTT:
                return "1xRTT"; // ~ 50-100 kbps
            case TelephonyManager.NETWORK_TYPE_CDMA:
                return "CDMA"; // ~ 14-64 kbps
            case TelephonyManager.NETWORK_TYPE_EDGE:
                return "EDGE"; // ~ 50-100 kbps
            case TelephonyManager.NETWORK_TYPE_EVDO_0:
                return "EVDO_0"; // ~ 400-1000 kbps
            case TelephonyManager.NETWORK_TYPE_EVDO_A:
                return "EVDO_A"; // ~ 600-1400 kbps
            case TelephonyManager.NETWORK_TYPE_GPRS:
                return "GPRS"; // ~ 100 kbps
            case TelephonyManager.NETWORK_TYPE_HSDPA:
                return "HSDPA"; // ~ 2-14 Mbps
            case TelephonyManager.NETWORK_TYPE_HSPA:
                return "HSPA"; // ~ 700-1700 kbps
            case TelephonyManager.NETWORK_TYPE_HSUPA:
                return "HSUPA"; // ~ 1-23 Mbps
            case TelephonyManager.NETWORK_TYPE_UMTS:
                return "UMTS"; // ~ 400-7000 kbps
            /*
             * Above API level 7, make sure to set android:targetSdkVersion 
             * to appropriate level to use these
             */
            //case TelephonyManager.NETWORK_TYPE_EHRPD: // API level 11 
            case 14:
                return "EHRPD"; // ~ 1-2 Mbps
            case TelephonyManager.NETWORK_TYPE_EVDO_B: // API level 9
                return "EVDO_B"; // ~ 5 Mbps
            //case TelephonyManager.NETWORK_TYPE_HSPAP: // API level 13
            case 15: // TelephonyManager.NETWORK_TYPE_HSPAP
                return "HSPAP"; // ~ 10-20 Mbps
            case TelephonyManager.NETWORK_TYPE_IDEN: // API level 8
                return "IDEN"; // ~25 kbps 
            //case TelephonyManager.NETWORK_TYPE_LTE: // API level 11
            case 13: // TelephonyManager.NETWORK_TYPE_LTE
                return "LTE"; // ~ 10+ Mbps
            // Unknown
            case TelephonyManager.NETWORK_TYPE_UNKNOWN:
            default:
                return "UNKNOWN";
        }
    }

    public static String getProvidersName(Context context) {
        try {
            TelephonyManager telephonyManager = (TelephonyManager) context.getSystemService(Context.TELEPHONY_SERVICE);
            String operator = telephonyManager.getSimOperator();
            if (operator == null || operator.equals("")) {
                return "unknown";
            }

            if (operator.startsWith("46000") || operator.startsWith("46002")) {
                return "cm";
            } else if (operator.startsWith("46001")) {
                return "cu";
            } else if (operator.startsWith("46003")) {
                return "ct";
            } else {
                return "unknown";
            }
        } catch (Exception e) {
            return "error";
        }
    }

    public static String osInfoBoardStatic() {
        mContext = AppActivity.getContext();
        String info = android.os.Build.MODEL + "|Android|" + android.os.Build.VERSION.RELEASE + "|" + AppActivity.getProvidersName(mContext) + "|";
        String str = "";

        if (!AppActivity.isConnected(mContext)) {
            return info + "Disconnected";
        } else {
            str = AppActivity.getWifi(mContext);
            if (str != "") {
                return info + str;
            }

            str = AppActivity.getMobile(mContext);
            if (str != "") {
                return info + str;
            } else {
                return info + "Unknown";
            }
        }
    }
}
```

因为判断4G的TelephonyManager.NETWORK_TYPE_LTE属于API level 11，用这个定义写法，编译无法通过（默认编译是API level 10，最低可以是API level 9），所以这里写了硬编码。

## 3. 配置文件修改

------

修改项目目录frameworks/runtime-src/proj.android/jni/Android.mk文件，加上jsb_os_info.hpp：

```
...
LOCAL_SRC_FILES := hellojavascript/main.cpp \
                   ../../Classes/jsb_os_info.hpp \
                   ../../Classes/AppDelegate.cpp
...
```

然后重新编译就可以在jsb环境下，用javascript执行osInfo这个函数。

## 4. javascript端执行效果

------

Wifi和手机上网状态，用javascript的osInfo函数返回分别是这样的：

```
vivo Xplay3S|Android|4.3|cm|Wifi|4|39
vivo Xplay3S|Android|4.3|cm|Mobile|LTE
```

主要是游戏对网络比较敏感，所以判断了是Wifi上网还是手机上网，Wifi还判断了信号强度以及连接速率，手机上网判断是哪种上网方式。

## 5. 反射机制调用

------

就在文档写到尾声的时候，在英文论坛突然发现这篇文档：

https://github.com/joshuastray/cocos-docs/blob/master/manual/framework/html5/v3/reflection/zh.md

有点无语，白白多花了几个小时，原来世界这么简单。强烈呼吁cocos2d-js团队的童鞋有文档就先在论坛里扔一下，小白鼠们都会抢着去测试的。

PS. IOS下javascript调用Objective-C有没有方便的方法？还是得普通jsb方式调用？

## 6. 参考文档

------

【集成友盟社会化SDK】COCOS2D-JS3.0ALPHA2中JAVA/C++/JAVASCRPIT 三方交互的实现
* http://www.asmld.com/?p=113

3.jni_c++调用java中的方法
* http://banshaotang.iteye.com/blog/2078501

cocos2d-x 使用JniHelper 调用 java代码 获取安卓生成的唯一标示UUID
* http://blog.csdn.net/yxiaom/article/details/17137479

