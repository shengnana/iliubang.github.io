---
layout: post
title: Java Native Interface（三）
subtitle: JNI练习
author: 刘邦
date: "2017-03-20 22:30:12"
weather: sunny
catalog: true
category: Java
tags: [Java, JNI]
comments: true
---


前面系统研究了JNI的相关操作，下面就来小试牛刀，做一个实际的练习。

记得去年我曾经用C语言写过一个PHP的md5扩展函数，那么今天就花一点点时间用JNI来实现一遍吧。

不过这里可要提前声明了，虽然是实现md5函数，但是这里并不会从头写md5算法，而是投机取巧使用到了linux内核提供的`crypto`库。

废话不多说，首先来写一个Java类

MyString.java

```java
public class MyString {
    static {
        System.loadLibrary("mymd5");
    }

    private String value;

    public native String md5();

    public MyString(String value) {
        this.value = value;
    }
}
```

然后生成头文件，并实现c代码：

```c
#include <jni.h>
#include <openssl/md5.h> 
#include <string.h>

JNIEXPORT jstring JNICALL Java_MyString_md5(JNIEnv *env, jobject obj) {
    jclass thisClass = (*env)->GetObjectClass(env, obj);
    jfieldID fidValue = (*env)->GetFieldID(env, thisClass, "value", "Ljava/lang/String;");
    if (NULL == fidValue) {
        return NULL;
    }
    jstring value = (*env)->GetObjectField(env, obj, fidValue);

    const unsigned char *data = (*env)->GetStringUTFChars(env, value, NULL);
    MD5_CTX ctx;
    unsigned char md[16];
    char buf[33]= {'\0'};
    char tmp[3]= {'\0'};
    int i;
    MD5_Init(&ctx);
    MD5_Update(&ctx,data,strlen((char *)data));
    MD5_Final(md,&ctx);

    for( i=0; i<16; i++ ) {
        sprintf(tmp,"%02X",md[i]);
        strcat(buf,tmp);
    }

    value = (*env)->NewStringUTF(env, buf);
    return value;
}
```

至此代码平淡无奇，也没什么好解释的，不过需要注意的是编译成动态链接库的部分，由于这里依赖到了其他的动态链接库，所以编译参数需要使用`-L{path} -l{libname}`来显示的指明依赖库的路径和库名

```
gcc -fPIC -shared -I /opt/app/java/include/ -I /opt/app/java/include/linux/ -I /usr/include/openssl/ -L/usr/lib/x86_64-linux-gnu/ MyString.c -lcrypto -o libmymd5.so
```

这样就编译好了，然后我们写一个测试类

```java
public class Test {
    public static void main(String[] args) {
        MyString s = new MyString("liubang");
        System.out.println(s.md5());   
    }
}
```

编译并运行：

```
ubuntu@vm-911:~/workspace/java/jni/demo4$ java -Djava.library.path=. Test
D67DF7EAE3E651E9DA077E9423C7A9B6
```

enjoy!

