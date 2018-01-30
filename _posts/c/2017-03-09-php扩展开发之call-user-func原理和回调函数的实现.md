---
layout: post
title: php扩展开发之call_user_func原理和回调函数的实现
author: 刘邦
excerpt: "很多时候，我们需要通过函数名来调用函数，并传递参数，或者把匿名函数作为函数的参数传递，实现回调......"
date: "2017-03-09 17:03:29"
catalog: true
tags: [c, php]
category: c
comments: true
---

# 函数调用

很多时候，我们需要通过函数名来调用函数，并传递参数，或者把匿名函数作为函数的参数传递，实现回调。当我们在遇到这样的需求的时候，用php代码实现起来肯定是非常容易和简单的。但是，当我们在用c语言编写php扩展的时候，如何来实现这样的功能呢？下面就一起来深入了解php内核，看看如何实现。

在Zend引擎中，给我们提供了`zend_call_function`,`call_user_function`以及`call_user_function_ex`函数来帮助我们实现函数调用。在`zend_API.h`文件中，我们可以看到如下函数原型的声明：

<!-- more -->
```c
ZEND_API int zend_call_function(zend_fcall_info *fci, zend_fcall_info_cache *fci_cache TSRMLS_DC);
ZEND_API int call_user_function(HashTable *function_table, zval **object_pp, zval *function_name, zval *retval_ptr, zend_uint param_count, zval *params[] TSRMLS_DC);
ZEND_API int call_user_function_ex(HashTable *function_table, zval **object_pp, zval *function_name, zval **retval_ptr_ptr, zend_uint param_count, zval **params[], int no_separation, HashTable *symbol_table TSRMLS_DC);
```
从函数的参数上来看，显然`zend_call_function`需要的参数很少，而其他两个都需要一堆参数，所以，我们可能会想，达到相同的效果为什么参数上有如此大的区别，于是带着这个疑问我们来解刨`zend_fcall_info`结构体。同样在`zend_API.h`中会看到如下结构体的定义：

```c
typedef struct _zend_fcall_info {
        size_t size;
        HashTable *function_table; //函数表
        zval *function_name;	   //函数，可以是函数名，也可以直接是匿名函数本身
        HashTable *symbol_table;   //符号表 
        zval **retval_ptr_ptr;	   //返回值
        zend_uint param_count;     //参数个数
        zval ***params; 		   //参数，数组
        zval *object_ptr;		   //调用对象方法时候需要传调用的对象
        zend_bool no_separation;   
} zend_fcall_info;
```
不难发现，原来是把相关字段封装到了结构体中了，所以显得参数数量少，实际上该有的都有。

对于`call_user_function`中相关参数含义的解释，这里就不强行翻译了，引用官方的一段话，虽然是英文，但是我相信聪明的你也肯定能看懂的！

> User functions can be called with the function call_user_function_ex(). It requires a hash value for the function table you want to access, a pointer to an object (if you want to call a method), the function name, return value, number of arguments, argument array, and a flag indicating whether you want to perform zval separation.
> Note that you don't have to specify both function_table and object; either will do. If you want to call a method, you have to supply the object that contains this method, in which case call_user_function()automatically sets the function table to this object's function table. Otherwise, you only need to specify function_table and can set object to NULL.
> Next is the parameter count as integer and an array containing all necessary parameters. The last argument specifies whether the function should perform zval separation - this should always be set to 0. If set to 1, the function consumes less memory but fails if any of the parameters need separation.

# 实现自己的`call_user_func`函数

废话不多说，下面就来动手实现一个自己的`call_user_func`函数;
这个函数是一个比较特殊的函数，因为他除了第一个参数是一个字符串之外，剩余的参数都是可变参数，而且没有固定的个数，所以想到这里是不是发现又遇到了一些小小的困难。不过没关系，遇到一切问题首先要想到查阅官方文档，于是在[php官方文档](http://php.net/manual/en/internals2.funcs.php)中找到了答案，在该文档页中，向我们列举了所有的Type Specifiers:

| Spec | Type | Locals |
|:-----|:-----|:-------|
| a | array | zval\* | 
| A | array or object | zval\* |
| b | boolean | zend_bool |
| C | class | zend_class_entry\* |
| d | double | double |
| f | function | zend_fcall_info\*, zend_fcall_info_cache\* |
| h | array | HashTable\* |
| H | array or object | HashTable\* |
| l | long | long |
| L | long(limits out-of-range LONG_MAX/LONG_MIN) | long |
| o | object | zval\* |
| O | object(of specified zend_class_entry) | zval\*, zend_class_entry \* |
| p | string(a valid path) | char\*, int |
| r | resource | zval\* |
| s | string | char\*, int |
| z | mixed | zval\* |
| Z | mixed | zval\*\* |

当我们看到这张表的时候只是了解到了在php扩展中参数传递对应关系，以及如何切当使用类型标记符，但是并没有解决我们需要传递不定长参数的问题。别着急，往下看会看到 Advanced Type Specifiers这样的文字，是的，你没有看错，这玩意还有高级用法：

| Spec | Description |
|:-----|:------------|
| \* | a variable number of argument of the preceeding type, 0 or more |
| + | a variable number of argument of the preceeding type, 1 or more |
| $\mid$ | indicates that the remaining parameters are optional |
| ! | the preceeding parameter can be of the specified type or null For 'b', 'l' and 'd', an extra argument of type zend_bool\* must be passed after the corresponding bool\*, long\* or double\* addresses which will be set true if null is recieved. |

\*，这不正是我们需要的吗！万事俱备，只欠东风，接下来，实现自己的`call_user_func`就是顺水推舟的事情了。

```c
PHP_FUNCTION(demo_call_user_func)
{
	zend_fcall_info  fci;
	zend_fcall_info_cache fci_cache;
	zval *retval_ptr = NULL;
	if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "f*", &fci, &fci_cache, &fci.params, &fci.param_count) != SUCCESS) {
		return;
	}
	fci.retval_ptr_ptr = &retval_ptr;
	if (zend_call_function(&fci, &fci_cache TSRMLS_CC) == SUCCESS && fci.retval_ptr_ptr && *fci.retval_ptr_ptr) {
		return_value = *fci.retval_ptr_ptr;
		zval_copy_ctor(return_value);
		zval_ptr_dtor(&fci.retval_ptr_ptr);
	}
	if (fci.params) {
		efree(fci.params);
	}
}
```
看到这里的代码，可能有些人还是一头雾水，不太明白为什么要这么写，至此我需要声明一下，我在写这篇博客的时候假想读者都是对php扩展开发有过一定了解的人，至少知道如何用c语言写一个输出"hello world"的扩展，知道如何返回值，以及如何创建php中的基本数据类型，如何赋值甚至了解php扩展中如何开发一个类。所以呢，假如你还没有这些基础怎么办，那也没办法，建议先回去补充知识咯。

写好扩展，编译安装后我们来测试一下：

```php
<?php

function say($msg) {
	echo $msg, PHP_EOL;
}

demo_call_user_func('say', "hello world");

demo_call_user_func(function($msg) {	
	echo $msg, PHP_EOL;
}, "hello liubang");

```
测试结果很自然的如我们所期待：

```shell
ubuntu@vm-911:~/workspace/c/php-5.6.30/ext/demo$ php test.php 
hello world
hello liubang
```

# 回调函数

有了上面的铺垫，我想到这里写一个回调函数已经是一件非常简单的事情了，只不过在`call_user_func`之前或者之后来做一些其他的操作。废话不多说，直接上代码：

```c
PHP_FUNCTION(demo_callback)
{
	zend_fcall_info fci;
	zend_fcall_info_cache fci_cache;
	if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "f", &fci, &fci_cache) == FAILURE) {
		return;
	}

	php_printf("this is callback demo...\n");
	php_printf("start call callback function\n");
	zval *retval_ptr = NULL;
	fci.retval_ptr_ptr = &retval_ptr;
	zval **arg[1];
	zval *param;
	MAKE_STD_ZVAL(param);
	ZVAL_STRING(param, "hello world", 0);
	arg[0] = &param;
	fci.param_count = 1;
	fci.params = arg;
	if (zend_call_function(&fci, &fci_cache TSRMLS_CC) == SUCCESS && fci.retval_ptr_ptr && *fci.retval_ptr_ptr) {
		COPY_PZVAL_TO_ZVAL(*return_value, *fci.retval_ptr_ptr);
	}
}
```

以上代码所对应的php代码为：

```php
<?php

function demo_callback(callable $callback) {
	echo "this is callback demo...\n";
	return $callback("hello world");
}

```

下面我们编译安装后测试一下：

```php
<?php

demo_callback(function($msg) {
	echo $msg, PHP_EOL;
	echo "this is in callback function\n";
});

```

执行结果如下：

```bash
ubuntu@vm-911:~/workspace/c/php-5.6.30/ext/demo$ php test.php 
this is callback demo...
start call callback function
hello world
this is in callback function
```

下面我再来用`call_user_function_ex`把回调重新实现一遍，写法基本一样：

```c
PHP_FUNCTION(demo_callback_o)
{
	zval *func;
	if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "z", &func) == FAILURE) {
		return;
	}
	php_printf("before callback...\n");
	char *func_name;
	if (!zend_is_callable(func, 0, &func_name TSRMLS_CC)) {
		php_error_docref(NULL TSRMLS_CC, E_WARNING, "Function: '%s' is not callable", func_name);
		RETURN_FALSE;
	}
	zval *retval;
	zval **arg[1];
	zval *param;
	MAKE_STD_ZVAL(param);
	ZVAL_STRING(param, "hello liubang", 0);
	arg[0] = &param;
	if (call_user_function_ex(EG(function_table), NULL, func, &retval, 1, arg, 0, NULL TSRMLS_CC) != SUCCESS) {
		php_printf("callback complated!\n");
	}
	COPY_PZVAL_TO_ZVAL(*return_value, retval);
}
```

效果和上边是一样的，所以这里就不再测试了。

# 总结
看到这里，也许你会觉得，这东西到底有什么用呢，是的，单纯地这么来看确实没有用，可是如果你也是一个对开源代码感兴趣的人，也许你也发现了在php著名的开源框架`swoole`中，就有很多地方使用到了类似的技巧，不同的是，`swoole`中是把所有的回调函数注册到一张表(也就是一个zval的数组)中，在请求的恰当阶段再来调用，于是就有了我们看到的类似于事件驱动的效果。所以，有了这些知识，假设你已经掌握了linux系统下的TCP/IP网络编程，那么你创造一个类似于`swoole`的产品又有何难。


