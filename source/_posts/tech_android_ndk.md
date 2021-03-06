---
layout: post
title: "Android studio NDK编译及so库生成方法讲解"
date: 6/30/2017 7:48:17 PM 
comments: true
tags: 
	- 技术 
	- Android
	- NDK
	- JNI C/C++
---
---
前言：在Android开发的eclipse时代，想要开发NDK项目或生成so库，是非常蛋疼的，需要踩坑无数，方能生成so库；而如今Android Studio时代，开发jni C/C++项目，通过gradle的集成工具，那是一个爽。下面将会介绍两种利用AS和gradle开发NDK项目及生成so库的方式。
## 一.环境准备
Android开发环境，Android-SDK，java-SDK,android-NDK相关环境（略：网上有许多）

**1.安装完成之后如图：**

![](/assets/img/ndk_config.png)

**2.在项目的gradle.properties文件中加上 android.useDeprecatedNdk = true**

**3.注意写好native接口和System.loadLibrary()**

如：JNIUtil和MainActivity

```java 
public class JNIUtil {

    private static JNIUtil instance = new JNIUtil();

    public static JNIUtil getInstance() {
        return instance;
    }

    static {
        System.loadLibrary("native-lib");
    }

    public native String initData();
    public native String getStringFromJni();
}

public class MainActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        TextView tv = (TextView) findViewById(R.id.sample_text);

        tv.setText(JNIUtil.getInstance().getStringFromJni());
        Log.i("笔沫拾光",JNIUtil.getInstance().initData());
        Toast.makeText(this,JNIUtil.getInstance().initData(),Toast.LENGTH_LONG);
    }
}

```
<!-- more -->
## 二.两种编译和生成SO库的方式

** 1.手动编译C/C++文件和so库生成 **

**i.生成C/C++文件**

执行Build->Make Project，生成class文件，class文件的生成路径为： app_path/build/intermediates/classes/debug.

javah生成c头文件

操作命令：
javah -d jni -classpath SDK_android.jar;APP_classes lab.sodino.jnitest.MainActivity

代码示例：

```java
javah -d jni -classpath D:\sdk\platforms\android-25\android.jar;E:\githup\JniTest\app\build\intermediates\classes\debug com.awen.jnitest.JNIUtil
```
生成头文件如图：

![](/assets/img/ndk_build_headfile.png)



**ii.编辑C文件**
在main.c文件中实现头文件中的方法

代码示例C：

```C

#include "com_awen_jnitest_JNIUtil.h"

JNIEXPORT jstring JNICALL Java_com_awen_jnitest_JNIUtil_initData(JNIEnv *env, jobject jObj){
    return (*env)->NewStringUTF(env, "有梦为马，随处可栖。");
}

JNIEXPORT jstring JNICALL Java_com_awen_jnitest_JNIUtil_getStringFromJni
        (JNIEnv *env, jobject jObj){
    return (*env)->NewStringUTF(env, "笔沫拾光\nhttp://awenzeng.me/");
}

```

代码示例C++：

```C++
#include <jni.h>
#ifdef __cplusplus
extern "C" {
#endif

jstring  Java_com_awen_jnitest_JNIUtil_initData(JNIEnv *env, jobject jObj){
    return env->NewStringUTF("Hello NDK C++");
}

jstring  Java_com_awen_jnitest_JNIUtil_getStringFromJni
        (JNIEnv *env, jobject jObj){
    return env->NewStringUTF("笔沫拾光\nhttp://awenzeng.me/");
}

#ifdef __cplusplus
}
#endif
```
**iii.修改build.gradle配置**

代码示例：

```java
  ndk {
            moduleName "native-lib"
            abiFilters "armeabi", "armeabi-v7a", "x86"//控制so库生成兼容的平台
  }
```

具体如图

![](/assets/img/ndk_gradle_config.png)

**iV.执行Build->Rebuild Project或Make Project，so库就会自动生成，具体如图：**

![](/assets/img/ndk_build_so.png)

** 2.Android studio配置工具编译和生成so库**

**i.Android studio工具配置**

为了方便生成头文件和so文件，我们可以在Android Studio → External Tools中设置两个命令，分别来生成头文件和生成.so文件
![](/assets/img/ndk_tools.png)

javah:

![](/assets/img/ndk_tool_javah.png)

具体配置代码：

```java
Program:
 $JDKPath$/bin/javah

Parameters:(具体参数参考第一种方法头文件的生成)
 -d jni -classpath D:\sdk\platforms\android-25\android.jar;E:\githup\JniTest\app\build\intermediates\classes\debug $FileClass$

Working
 $SourcepathEntry$\..\java

```

ndk-build:

![](/assets/img/ndk_tool_ndk_build.png)

具体配置代码：

```java
Program:
 D:\sdk\ndk-bundle\build\ndk-build.cmd

Parameters:
 NDK_LIBS_OUT=$ModuleFileDir$/libs APP_ABI=armeabi-v7a,armeabi,x86

Working
$ModuleFileDir$\src\main

```

**ii.C/C++文件生成及so库生成**

**头文件.h的生成，具体操作如图：**

![](/assets/img/ndk_tool_gen_headfile.png)

具体步骤：

选中JNIUtil点击右键，显示如上图，选中NDK，点击javah，就会自动生成头文件，具体位置如图：

![](/assets/img/ndk_build_headfile.png)

**.c文件生成和第一种一样（这里略）**

**so库文件生成，点击ndk-build，生成库文件，具体如图：**

![](/assets/img/ndk_ndk_build_so.png)


到此，两种方法生成讲解完毕。

## 三.注意事项

**第一种方式**
- gradle.build中生成so库文件的平台可配置，如（具体如第一种方式build配置）：abiFilters "armeabi", "armeabi-v7a", "x86"//控制so库生成兼容的平台
- 生成的so库是在build文件中，需要手动copy到项目。

**第二种方式**
- 在ndk-build的配置中，so库文件生成平台也可以配置，如（具体如ndk-build配置）：APP_ABI=armeabi-v7a,armeabi,x86
- so库文件的生成位置是可以配置的，自动生成到你配置的位置（这一点爽爆了，不用copy）,如（具体如ndk-build配置）：NDK_LIBS_OUT=$ModuleFileDir$/libs
- 可以随时修改C/C++代码，然后点击ndk-build生成so库，直接调试，非常方便。



#### 源码地址，欢迎下载及Star 
[https://github.com/awenzeng/JniTest](https://github.com/awenzeng/JniTest)

## 四.参考文献
[android studio NDK使用，编译c生成.so实践记录](http://blog.csdn.net/u010030505/article/details/51942157)

[Android Studio开发JNI工程](http://blog.csdn.net/sodino/article/details/41946607)

[Android NDK Jni 开发C和C++的区别](http://www.cnblogs.com/gengchangjing/p/ndk.html)






