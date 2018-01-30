---
layout: post
title: Java Native Interface（二）
author: 刘邦
date: "2017-03-20 18:10:24"
excerpt: "JNI进阶"
catalog: true
category: Java
tags: [Java, JNI]
comments: true
---

# JNI基础

JNI中定义了一下类型来对应到相应的Java的数据类型:

**1. Java基本数据类型:** `jint`,`jbyte`,`jshort`,`jlong`,`jfloat`,`jdouble`,`jchar`,`jboolean`分别对应Java中的`int`,`byte`,`short`,`long`,`float`,`double`,`char`和`boolean`。

**2. Java引用类型：**
`jobject`对应`java.lang.object`。同时也定义了下列子类型：

- `jclass`对应`java.lang.Class`
- `jstring`对应`java.lang.String`
- `jthrowable`对应`java.lang.Throwable`
- `jarray`对应Java中的数组。Java中的数组由8种基本数据类型和一个`Object`类型派生二来，所以JNI中也存在`jintArray`,`jbyteArray`,`jshortArray`,`jlongArray`,`jfloatArray`,`jdoubleArray`,
`jcharArray`,`jbooleanArray`和`jobjectArray`

native函数接收和返回上述的JNI类型数据。如果native函数需要操作它自己的数据类型(如c语言中的int, char *)，那么就需要在JNI类型和本地类型之间做相应的转换。

简而言之，native函数的编写流程大致为：

1. 通过Java程序接收JNI类型的参数
2. 将接收的JNI类型转换成本地类型
3. 完成相应的操作
4. 创建一个需要返回的JNI类型的对象，然后将返回的数据copy到要返回的对象中
5. 返回

从上述流程可以看出，编写JNI程序主要的挑战在于数据类型之间的转换，然而JNI中提供了很多转换函数来帮助我们完成相应的操作。

JNI是一个c语言的接口，c语言并不支持OOP的特性(严格的说，OOP是一种理念，这里只是从语言本身来说c语言不支持面向对象，实际上用c语言也可以写出面向对象风格的程序！)，所以他们之间并不是真的通过对象来传递。

# 在Java程序和native程序之间传递参数

## 传递基本类型

传递Java中的基本数据类型是非常简单的。一个`jxxx`类型被定义在native环境中，直接对应着Java中`xxx`基本类型。

```java
public class TestAdd {
	static {
		System.loadLibrary("myadd");
	}

	private native double add(int a, int b);

	public static void main(String[] args) {
		System.out.println("In Java, the summary is " + new TestAdd().add(3, 2));
	}
}
```

编译Java并生成C/C++头文件

```
javac TestAdd.java
javah TestAdd
```

实现C代码

```c
#include <stdio.h>
#include <jni.h>
#include "TestAdd.h"

JNIEXPORT jdouble JNICALL Java_TestAdd_add  (JNIEnv *env, jobject this, jint a, jint b) {
	jint	result;
	printf("In c, the numbers are %d and %d\n", a, b);
	result = (jint)(a + b);
	return result;
}
```

将C代码编译成动态链接库

```
gcc -fPIC -shared -I /opt/app/java/include/ -I /opt/app/java/include/linux/ TestAdd.c -o libmyadd.so
```

运行Java程序

```
java -Djava.library.path=. TestAdd

====output====
ubuntu@vm-911:~/workspace/java/jni/demo2$ java -Djava.library.path=. TestAdd
In c, the numbers are 3 and 2
In Java, the summary is 5.0
```

上面的例子是传递整型类型的情况，比较简单，不需要做转换处理，那么下面我们来看看传递字符串类型的情况。

## 传递String

由于编译方式几乎每次都一样，所以接下来的例子只呈现代码，不再描述具体的编译操作，如果有问题请阅读Java Native Interface（一）。

```java
public class TestJNIString {
    static {
        System.loadLibrary("myjnistring");
    }

    private native String say(String msg);

    public static void main(String[] args) {
        String result = new TestJNIString().say("hello liubang");
        System.out.println("In Java, the result is " + result);
    }
}
```

实现C逻辑

```c
#include <stdio.h>
#include <jni.h>
#include "TestJNIString.h"

JNIEXPORT jstring JNICALL Java_TestJNIString_say(JNIEnv *env, jobject this, jstring msg) {
    const char *cString = (*env)->GetStringUTFChars(env, msg, NULL);
    if (NULL == cString) {
        return NULL;
    }

    printf("In c, the received string is %s\n", cString);
    (*env)->ReleaseStringUTFChars(env, msg, cString);

    char outPut[128];
    printf("Enter a String: ");
    scanf("%s", outPut);

    return (*env)->NewStringUTF(env, outPut);
}
```

由于上述代码太简单，所以这里不做具体的解释，关于相关函数的原型，可以在`jni.h`头文件中找到定义。

测试结果如下：

```
ubuntu@vm-911:~/workspace/java/jni/demo2$ java -Djava.library.path=. TestJNIString
In c, the received string is hello liubang
Enter a String: liubang
In Java, the result is liubang
```

## 传递数组

TestJNIArray.java

```java
public class TestJNIArray {
    static {
        System.loadLibrary("myjniarray");
    }

    private native double[] avg(int[] numbers);

    public static void main(String[] args) {
        int[] numbers = {12, 25, 8};
        double[] results = new TestJNIArray().avg(numbers);
        System.out.println("In Java the sum is " + results[0]);
        System.out.println("In java the avg is " + results[1]);
    }
}
```

TestJNIArray.c

```c
#include <jni.h>
#include "TestJNIArray.h"

JNIEXPORT jdoubleArray JNICALL Java_TestJNIArray_avg(JNIEnv *env, jobject this, jintArray jniarray) {
    jint *cArray = (*env)->GetIntArrayElements(env, jniarray, NULL);
    if (NULL == cArray)
        return NULL;
    
    jsize length = (*env)->GetArrayLength(env, jniarray);
    jint sum = 0;
    int i;
    for (i = 0; i < length; i++) {
        sum += cArray[i];
    }
    jdouble avg = (jdouble)sum / length;
    (*env)->ReleaseIntArrayElements(env, jniarray, cArray, 0);

    jdouble output[] = {sum, avg};
    jdoubleArray outJNIArray = (*env)->NewDoubleArray(env, 2);
    if (NULL == outJNIArray)
        return NULL;

    (*env)->SetDoubleArrayRegion(env, outJNIArray, 0, 2, output);
    return outJNIArray;
}
```

运行结果为：

```
ubuntu@vm-911:~/workspace/java/jni/demo2$ java -Djava.library.path=. TestJNIArray
In Java the sum is 45.0
In java the avg is 15.0
```

# 访问对象的成员变量和回调函数

## 访问成员变量

TestObjVariable.java

```java
public class TestObjVariable {
    static {
        System.loadLibrary("libmytestobjvariable");
    }

    private String name = "liubang";
    private int age = 24;

    private native void setNameAndAge();

    public static void main(String[] args) {
        TestObjVariable test = new TestObjVariable();
        test.setNameAndAge();
        System.out.println("My name is " + test.name + " and I am " + test.age + " years old");
    }    
}
```

C语言中的实现：

TestObjVariable.c

```c
#include <stdio.h>
#include <jni.h>
#include "TestObjVariable.h"

JNIEXPORT void JNICALL Java_TestObjVariable_setNameAndAge(JNIEnv *env, jobject obj) {
    //获取当前对象的Class
    jclass thisClass = (*env)->GetObjectClass(env, obj);

    jfieldID fidAge = (*env)->GetFieldID(env, thisClass, "age", "I");
    if (NULL == fidAge) {
        return;
    }

    jint age = (*env)->GetIntField(env, obj, fidAge);
    printf("In C, the age is %d\n", age);
    age = 22;
    (*env)->SetIntField(env, obj, fidAge, age);

    jfieldID fidName = (*env)->GetFieldID(env, thisClass, "name", "Ljava/lang/String;");
    if (NULL == fidName) {
        return;
    }

    jstring name = (*env)->GetObjectField(env, obj, fidName);
    const char *str = (*env)->GetStringUTFChars(env, name, NULL);
    if (NULL == str) {
        return;
    }

    printf("In C, the name is %s\n", str);
    (*env)->ReleaseStringUTFChars(env, name, str);

    name = (*env)->NewStringUTF(env, "钟灵");
    if (NULL == name) {
        return;
    }
    (*env)->SetObjectField(env, obj, fidName, name);
}
```

运行结果为：

```
ubuntu@vm-911:~/workspace/java/jni/demo2$ java -Djava.library.path=. TestObjVariable 
In C, the age is 24
In C, the name is liubang
My name is 钟灵 and I am 22 years old
```

可能在之前的例子中你会觉得平淡无奇，那么写到这里是不是发现有点意思了。

首先通过`GetObjectClass`函数来获取当前对象的类，然后通过`GetFieldID`函数来获取成员变量的ID，这里你需要提供字段的名字和字段类型的描述符。对于一个Java类类型的描述符，通常用"L<fully-qualified-name>;"的形式来描述，这里千万要注意最后有一个分号，例如上述例子中的对于String类型的描述符为"Ljava/lang/String;"。而对于基本数据类型，可以使用"I"来描述"int"，"B"来描述"byte"，"S"来描述"short"，"J"描述"long"，"F"描述"float"，"D"描述"double"，"C"描述"char"，"Z"描述"boolean"。如果是数组，则只需要加上一个"["的前缀即可，例如："[Ljava/lang/String;"描述一个String类型的数组。看到这里，假如你恰好也是一个PHP程序员，而且对PHP内核有所了解的话，是不是发现跟鞋PHP扩展时候参数的解析有一些类似呢。

基于字段ID，通过`GetObjectField`函数或者`Get<primitive-type>Field`函数来获取实例对象的成员变量。

更新实例对象的成员变量，则调用`SetObjectField`或者`Set<primitive-type>Field`函数。

## 访问静态成员

访问静态成员同访问成员变量很相似，不同的是使用的JNI函数为`GetStaticFieldID`, `Get|SetStaticObjectField`, `Get|SetStatic<Primitive-type>Field`。

TestObjStaticVariable.java

```java
public class TestObjStaticVariable {
    static {
        System.loadLibrary("libtestobjstaticvariable");
    }

    private static int age = 24;
    private static String name = "liubang";

    private native void setNameAndAge();

    public static void main(String[] args) {
        TestObjStaticVariable test = new TestObjStaticVariable();
        test.setNameAndAge();
        System.out.println("In java, my name is " + name + ", I am " + age + " years old");
    }
}
```

TestObjStaticVariable.c

```c
#include <stdio.h>
#include <jni.h>
#include "TestObjStaticVariable.h"

JNIEXPORT void JNICALL Java_TestObjStaticVariable_setNameAndAge(JNIEnv *env, jobject obj) {
    jclass class = (*env)->GetObjectClass(env, obj);

    jfieldID fidAge = (*env)->GetStaticFieldID(env, class, "age", "I");
    if (NULL == fidAge) {
        return;
    }
    
    jint age = (*env)->GetStaticIntField(env, obj, fidAge);
    printf("In C, the age is %d\n", age);

    age = 22;
    (*env)->SetStaticIntField(env, class, fidAge, age);
}
```

## 调用实例方法和静态方法

TestInstanceMethod.java

```java
public class TestInstanceMethod {
    static {
        System.loadLibrary("testinstance");
    }

    private native void foo();

    private void bar() {
        System.out.println("hello liubang");
    }

    private void bar(String message) {
        System.out.println("hello " + message);
    }

    private double avg(int a, int b) {
        return ((double)a + b)/2.0;
    }

    private static String s() {
        return "hello zhongling";
    }

    public static void main(String[] args) {
        new TestInstanceMethod().foo();
    } 
}
```

TestInstanceMethod.c

```c
#include <stdio.h>
#include <jni.h>
#include "TestInstanceMethod.h"

JNIEXPORT void JNICALL Java_TestInstanceMethod_foo(JNIEnv *env, jobject obj) {

    jclass thisClass = (*env)->GetObjectClass(env, obj);
    jmethodID midBar = (*env)->GetMethodID(env, thisClass, "bar", "()V");
    if (NULL == midBar) {
        return;
    }

    printf("In C, call Java's bar()\n");
    (*env)->CallVoidMethod(env, obj, midBar);

    jmethodID midBarStr = (*env)->GetMethodID(env, thisClass, "bar", "(Ljava/lang/String;)V");
    if (NULL == midBarStr) {
        return;
    }
    printf("In C, call Java's s()\n");
    jstring message = (*env)->NewStringUTF(env, "钟灵");
    (*env)->CallVoidMethod(env, obj, midBarStr, message);

    jmethodID midAvg = (*env)->GetMethodID(env, thisClass, "avg", "(II)D");
    if (NULL == midAvg) {
        return;
    }
    jdouble average = (*env)->CallDoubleMethod(env, obj, midAvg, 2, 5);
    printf("In C, the average of 2 and 5 is %f\n", average);

    jmethodID midStatic = (*env)->GetStaticMethodID(env, thisClass, "s", "()Ljava/lang/String;");
    if (NULL == midStatic) {
        return;
    }

    jstring res = (*env)->CallStaticObjectMethod(env, thisClass, midStatic);
    const char *cStr = (*env)->GetStringUTFChars(env, res, NULL);
    if (NULL == cStr) {
        return;
    }
    printf("In C, the returned string is %s\n", cStr);
    (*env)->ReleaseStringUTFChars(env, res, cStr);
}
```

测试结果为：

```
ubuntu@vm-911:~/workspace/java/jni/demo3$ java -Djava.library.path=. TestInstanceMethod
In C, call Java's bar()
hello liubang
In C, call Java's s()
hello 钟灵
In C, the average of 2 and 5 is 3.500000
In C, the returned string is hello zhongling
```

**注意：**也许细心的你已经发现了，在调用对象成员和静态成员(类成员)的时候，除了调用的函数中多了"static"之外，函数的参数也有所区别，对象成员中传入的是当前的对象实例，而静态成员传的则是当前的类。

上述代码非常简单，主要使用`GetMethodID`函数来获取想要调用的成员方法的methodID，这里需要解释的一点就是，对于函数的参数和返回值使用的是`(参数)返回值`的形式来做描述符，这里的类型描述符跟前面讲到获取对象成员时候的规则是一样的，这里不再赘述。

## 创建对象和对象数组

### 使用构造方法创建对象

TestCreateObj.java

```java
public class TestCreateObj {
    static {
        System.loadLibrary("createobj");
    }

    private native Integer getIntegerObj(int n);

    public static void main(String[] args) {
        TestCreateObj t = new TestCreateObj();
        System.out.println("In Java, the number is :" + t.getIntegerObj(1000));
    }
}
```

TestCreateObj.c

```c
#include <stdio.h>
#include <jni.h>
#include "TestCreateObj.h"

JNIEXPORT jobject JNICALL Java_TestCreateObj_getIntegerObj(JNIEnv *env, jobject obj, jint number) {
    jclass clazz = (*env)->FindClass(env, "java/lang/Integer");
    jmethodID midCons = (*env)->GetMethodID(env, clazz, "<init>", "(I)V");

    jobject intObj = (*env)->NewObject(env, clazz, midCons, number);
    jmethodID midToString = (*env)->GetMethodID(env, clazz, "toString", "()Ljava/lang/String;");
    if (NULL == midToString) {
        return NULL;
    }

    jstring resStr = (*env)->CallObjectMethod(env, intObj, midToString);
    const char *cStr = (*env)->GetStringUTFChars(env, resStr, NULL);
    printf("In C: the number is %s\n", cStr);

    return intObj;
}
```

运行结果为：

```
ubuntu@vm-911:~/workspace/java/jni/demo3$ java -Djava.library.path=. TestCreateObj
In C: the number is 1000
In Java, the number is :1000
```

## 创建对象数组

TestObjArr.java

```java
public class TestObjArr {
    static {
        System.loadLibrary("objarr");
    }

    private native Double[] sumAndAverage(Integer[] number);

    public static void main(String args[]) {
        Integer[] numbers = {11, 24, 88};
        Double[] results = new TestObjArr().sumAndAverage(numbers);
        System.out.println("In Java, the sum is " + results[0]);
        System.out.println("In Java, the average is " + results[1]);
    }
}
```

TestObjArr.c

```c
#include <stdio.h>
#include <jni.h>
#include "TestObjArr.h"

JNIEXPORT jobjectArray JNICALL Java_TestObjArr_sumAndAverage(JNIEnv *env, jobject obj, jobjectArray numbers) {
    jclass classInt = (*env)->FindClass(env, "java/lang/Integer");
    jmethodID midIntVal = (*env)->GetMethodID(env, classInt, "intValue", "()I");
    if (NULL == midIntVal) {
        return NULL;
    }

    jsize length = (*env)->GetArrayLength(env, numbers);
    jint sum = 0;
    int i;
    for (i = 0; i < length; i++) {
        jobject objInt = (*env)->GetObjectArrayElement(env, numbers, i);
        if (NULL == objInt) {
            return NULL;
        }
        jint value = (*env)->CallIntMethod(env, objInt, midIntVal);
        sum += value;
    }

    double average = (double)sum / length;
    printf("In C, the sum is %d\n", sum);
    printf("In C, the average is %f\n", average);

    jclass classDouble = (*env)->FindClass(env, "java/lang/Double");
    jobjectArray outArr = (*env)->NewObjectArray(env, 2, classDouble, NULL);
    jmethodID midDoubleInit = (*env)->GetMethodID(env, classDouble, "<init>", "(D)V");
    if (NULL == midDoubleInit) {
        return NULL;
    }

    jobject objSum = (*env)->NewObject(env, classDouble, midDoubleInit, (double)sum);
    jobject objAvg = (*env)->NewObject(env, classDouble, midDoubleInit, average);

    (*env)->SetObjectArrayElement(env, outArr, 0, objSum);
    (*env)->SetObjectArrayElement(env, outArr, 1, objAvg);

    return outArr;
}
```

测试结果为：

```
ubuntu@vm-911:~/workspace/java/jni/demo3$ java -Djava.library.path=. TestObjArr
In C, the sum is 123
In C, the average is 41.000000
In Java, the sum is 123.0
In Java, the average is 41.0
```

**注意：** 在JNI中调用构造方法的时候，方法名称用的是`<init>`

