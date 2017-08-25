---
layout: post
title: php扩展开发之打造一个简易的ArrayBuffer
subtitle: 快乐源于实干
author: 刘邦
weather: sunny
catalog: true
tags: [c,php]
category: c
use_math: true
---

## ArrayBuffer简介

ArrayBuffer又叫二进制数组，是一个用来表示通用的，固定长度的二进制数据缓冲区。你不能直接操纵ArrayBuffer的内容，
而是创建一个表示特定格式的buffer的类型化数组对象（也叫做数据视图对象）来对buffer的内容进行读写操作。

我最早了解ArrayBuffer是从JavaScript开始的，具体的用法和api可以参考[JavaScript标准库－－ArrayBuffer](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer)

那么接下来，我们就给PHP扩展一个简单的ArrayBuffer，顺便巩固一下[php扩展开发之自定义对象的存储](https://iliubang.github.io/c/2017/08/24/php%E6%89%A9%E5%B1%95%E5%BC%80%E5%8F%91%E4%B9%8B%E8%87%AA%E5%AE%9A%E4%B9%89%E5%AF%B9%E8%B1%A1%E7%9A%84%E5%AD%98%E5%82%A8/)。

## 定义ArrayBuffer的数据结构和相关`handlers`

`ArrayBuffer`是一个非常简单的对象，它只需要申明并存储一个`buffer`和它的长度即可：

```c
typedef struct _buffer_object {
    zend_object std;
    void *buffer;
    size_t length;
} buffer_object;
```

接下来我们来实现它的`create`和`free` handlers，有了前面的基础，这个实现也是及其简单的：

```c
static void linger_array_buffer_free_object_storage(buffer_object *intern TSRMLS_DC)
{
	zend_object_std_dtor(&intern->std TSRMLS_CC);
	linger_efree(intern->buffer);
}

zend_object_value linger_array_buffer_create_object(zend_class_entry *class_type TSRMLS_DC)
{
	zend_object_value retval;
	buffer_object *intern = emalloc(sizeof(buffer_object));
	memset(intern, 0, sizeof(buffer_object));

	zend_object_std_init(&intern->std, class_type TSRMLS_CC);
	object_properties_init(&intern->std, class_type);

	retval.handle = zend_objects_store_put(
			intern,
			(zend_objects_store_dtor_t) zend_objects_destroy_object,
			(zend_objects_free_object_storage_t) linger_array_buffer_free_object_storage,
			NULL
			TSRMLS_CC
			);

	retval.handlers = &linger_array_buffer_handlers;
	return retval;
}
```

从上面的代码中可以看到，我们并没有在`create_object`中申请`buffer`的空间，而这步操作将会在`__construct`中来实现，因为`buffer`的长度会作为构造函数的参数传递过来。

```c
/* ArrayBuffer arginfo */
ZEND_BEGIN_ARG_INFO_EX(arginfo_buffer_ctor, 0, 0, 1)
	ZEND_ARG_INFO(0, length)
ZEND_END_ARG_INFO()

PHP_METHOD(linger_ArrayBuffer, __construct)
{
	buffer_object *intern;
	long length;
	zend_error_handling error_handling;

	zend_replace_error_handling(EH_THROW, NULL, &error_handling TSRMLS_CC);
	if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "l", &length) == FAILURE) {
		zend_restore_error_handling(&error_handling TSRMLS_CC);
		return;
	}

	zend_restore_error_handling(&error_handling TSRMLS_CC);
	if (length < 0) {
		zend_throw_exception(NULL, "Buffer length must be positive", 0 TSRMLS_CC);
		return;
	}
	intern = zend_object_store_get_object(getThis() TSRMLS_CC);
	intern->buffer = emalloc(length);
	intern->length = length;

	memset(intern->buffer, 0, length);
}
```

由于现在写的是物件导向风格的程式码，所以我们不再直接抛出`error`，而是通过`zend_throw_exception`抛`exception`，
它需要传递一个异常类的`class entry`，异常信息和错误码作为参数，如果异常类的`class entry`为`NULL`的话，就会
使用默认的`Exception`类抛出。

对于`__construct`这样的方法尤其需要注意，当发生错误的时候抛出异常能够避免一个对象被部分构造。这就是上述代码
替换掉错误处理模式的原因。通常情况下，`zend_parse_parameters`在解析非法参数的情况下只会抛出一个`warning`，通过
将错误模式设置为`EH_NORMAL`可以自动将警告转换成异常抛出。

使用`zend_replace_error_handling`函数可以改变错误处理模式。它的第一个参数为错误模式：`EH_NORMAL`、`EH_SUPPRESS`、`EH_THROW`，第二个参数为异常类的`class entry`，如果为`NULL`，则使用
默认的`Exception`作为异常类抛出，最后一个参数是一个指向`zend_error_handling`结构的指针，用以备份之前的`error mode`。

除了`create handler`之外，我们还需要处理克隆操作。

```c
static zend_object_value linger_array_buffer_clone(zval *object TSRMLS_DC)
{
	buffer_object *old_object = zend_object_store_get_object(object TSRMLS_CC);
	zend_object_value new_object_val = linger_array_buffer_create_object(Z_OBJCE_P(object) TSRMLS_CC);
	buffer_object *new_object = zend_object_store_get_object_by_handle(new_object_val.handle TSRMLS_CC);

	zend_objects_clone_members(&new_object->std, new_object_val, &old_object->std, Z_OBJ_HANDLE_P(object) TSRMLS_CC);

	new_object->buffer = old_object->buffer;
	new_object->length = old_object->length;

	if (old_object->buffer) {
		new_object->buffer = emalloc(old_object->length);
		memcpy(new_object->buffer, old_object->buffer, old_object->length);
	}
	memcpy(new_object->buffer, old_object->buffer, old_object->length);
	return new_object_val;
}
```

最后在`MINIT`中注册ArrayBuffer类：

```c
PHP_MINIT_FUNCTION(linger_array_buffer)
{
	zend_class_entry ce;
	INIT_CLASS_ENTRY(ce, "Linger\\ArrayBuffer", linger_array_buffer_methods);
	linger_array_buffer_ce = zend_register_internal_class(&ce TSRMLS_CC);
	linger_array_buffer_ce->create_object = linger_array_buffer_create_object;
	memcpy(&linger_array_buffer_handlers, zend_get_std_object_handlers(), sizeof(zend_object_handlers));
	linger_array_buffer_handlers.clone_obj = linger_array_buffer_clone;

	return SUCCESS;
}
```

## buffer view

下面我们来打造数据视图类。我们将要在同一个示例中实现8中不同的数据视图类，分别叫做`Int8Array`，`UInt8Array`，
`Int16Array`，`UInt16Array`，`Int32Array`，`UInt32Array`，`FloatArray`，`DoubleArray`。首先我们在`MINIT`中
注册这些类：

```c
/* ArrayBufferView class entry and handlers */
zend_class_entry *int8_array_ce;
zend_class_entry *uint8_array_ce;
zend_class_entry *int16_array_ce;
zend_class_entry *uint16_array_ce;
zend_class_entry *int32_array_ce;
zend_class_entry *uint32_array_ce;
zend_class_entry *float_array_ce;
zend_class_entry *double_array_ce;
zend_object_handlers linger_array_buffer_view_handlers;

PHP_MINIT_FUNCTION(linger_array_buffer)
{
	zend_class_entry ce;
	// ArrayBuffer class register
	...
#define REGISTER_ARRAY_BUFFER_VIEW_CLASS(class_name, type)		                 \
	do {														                 \
		INIT_CLASS_ENTRY(ce, #class_name, linger_array_buffer_view_methods);     \
		type##_array_ce = zend_register_internal_class(&ce TSRMLS_CC);           \
		type##_array_ce->create_object = linger_array_buffer_view_create_object; \
		zend_class_implements(type##_array_ce TSRMLS_CC, 1, zend_ce_arrayaccess);\
	} while (0)

	REGISTER_ARRAY_BUFFER_VIEW_CLASS(Linger\\ArrayBufferView\\Int8Array, int8);
	REGISTER_ARRAY_BUFFER_VIEW_CLASS(Linger\\ArrayBufferView\\UInt8Array, uint8);
	REGISTER_ARRAY_BUFFER_VIEW_CLASS(Linger\\ArrayBufferView\\Int16Array, int16);
	REGISTER_ARRAY_BUFFER_VIEW_CLASS(Linger\\ArrayBufferView\\UInt16Array, uint16);
	REGISTER_ARRAY_BUFFER_VIEW_CLASS(Linger\\ArrayBufferView\\Int32Array, int32);
	REGISTER_ARRAY_BUFFER_VIEW_CLASS(Linger\\ArrayBufferView\\UInt32Array, uint32);
	REGISTER_ARRAY_BUFFER_VIEW_CLASS(Linger\\ArrayBufferView\\FloatArray, float);
	REGISTER_ARRAY_BUFFER_VIEW_CLASS(Linger\\ArrayBufferView\\DoubleArray, double);

	memcpy(&linger_array_buffer_view_handlers, zend_get_std_object_handlers(), sizeof(zend_object_handlers));
	linger_array_buffer_view_handlers.clone_obj = linger_array_buffer_view_clone;
	return SUCCESS;
}
```

为了避免书写大量重复类似的代码，我们定义了一个宏来简化书写，注意，在macro中`#`表示将参数作为字符串传递，而`##`表示连接操作。

接下来我们来申明`linger_array_buffer_view_methods`：

```c
const zend_function_entry linger_array_buffer_view_methods[] = {
	PHP_ME_MAPPING(__construct, linger_array_buffer_view_ctor, arginfo_buffer_view_ctor, ZEND_ACC_PUBLIC | ZEND_ACC_CTOR)
	PHP_ME_MAPPING(offsetGet, linger_array_buffer_view_offset_get, arginfo_buffer_view_offset, ZEND_ACC_PUBLIC)
	PHP_ME_MAPPING(offsetSet, linger_array_buffer_view_offset_set, arginfo_buffer_view_offset_set, ZEND_ACC_PUBLIC)
	PHP_ME_MAPPING(offsetExists, linger_array_buffer_view_offset_exists, arginfo_buffer_view_offset, ZEND_ACC_PUBLIC)
	PHP_ME_MAPPING(offsetUnset, linger_array_buffer_view_offset_unset, arginfo_buffer_view_offset, ZEND_ACC_PUBLIC)
	PHP_FE_END
};
```

这里用到了一个很新鲜的玩意儿，`PHP_ME_MAPPING`。通常我们在定义`zend_function_entry`的时候都是使用`PHP_ME`宏，那么
`PHP_ME`和`PHP_ME_MAPPING`的区别是什么呢，下面我们来举例说明：

```c
PHP_ME(linger_ArrayBuffer_View, offsetGet, arginfo_buffer_view_offset, ZEND_ACC_PUBLIC)
/* 会映射到 */
PHP_METHOD(linger_ArrayBuffer_View, offsetGet) {...}

PHP_ME_MAPPING(offsetGet, linger_array_buffer_offset_get, arginfo_buffer_view_offset, ZEND_ACC_PUBLIC)
/* 会映射到 */
PHP_FUNCTION(linger_array_buffer_offset_get) {...}
```

至此，也许你也应该意识到了`PHP_FUNCTION`和`PHP_METHOD`其实什么都没有做，他们只是定义了一些函数的名字和参数而已。
这就是为什么你可以把一个`function`注册成`method`，你也可以定义一个方法的时候使用一个名字，而注册的时候使用另一个名字。
这对于支持物件导向接口和程式API都是非常有用的。

而回到我们的代码中，这里之所以选择使用`PHP_ME_MAPPING`是因为我们没有一个确切的`ArrayBufferView`类，而是一组公用方法的类。

接下来我们来思考`buffer view`的数据结构该如何组织：首先需要能够区分不同的视图类，例如一些标记字段，其次它还需要存储一个
`zval`类型的数据来记录它所操作的`ArrayBuffer`对象，最后需要一个成员能够作为不同的类型用来访问`ArrayBuffer`中的`buffer`。
此外它还需要记录当前的`offset`和`length`，因为我们在创建数据视图的时候不一定会使用到整个`ArrayBuffer`中的`buffer`，例如：
`new linger\ArrayBuffer\Int32Array($buffer, 8, 10)`，将会创建一个数据视图，从`ArrayBuffer`的偏移`sizeof(int32_t) * 8`bytes的位置开始，
总共包含24和元素。

下面是`Array Buffer View`的数据结构：

```c
typedef enum _buffer_view_type {
	buffer_view_int8,
	buffer_view_uint8,
	buffer_view_int16,
	buffer_view_uint16,
	buffer_view_int32,
	buffer_view_uint32,
	buffer_view_float,
	buffer_view_double,
} buffer_view_type;

typedef struct _buffer_view_object {
	zend_object std;
	zval *buffer_zval;
	union {
		int8_t    *as_int8;
		uint8_t   *as_uint8;
		int16_t   *as_int16;
		uint16_t  *as_uint16;
		int32_t   *as_int32;
		uint32_t  *as_uint32;
		float     *as_float;
		double    *as_double;
	} buf;
	size_t offset;
	size_t length;
	buffer_view_type type;
} buffer_view_object;
```

与`ArrayBuffer`类似，下面是`free`和`create_object`handler的定义，由于很简单，只给出代码，不再做详细的分析：

```c
static void linger_array_buffer_view_free_object_storage(buffer_view_object *intern TSRMLS_DC)
{
	zend_object_std_dtor(&intern->std TSRMLS_CC);
	if (intern->buffer_zval) {
		zval_ptr_dtor(&intern->buffer_zval);
	}
	linger_efree(intern);
}

zend_object_value linger_array_buffer_view_create_object(zend_class_entry *class_type TSRMLS_CC)
{
	zend_object_value retval;
	buffer_view_object *intern = emalloc(sizeof(buffer_view_object));
	memset(intern, 0, sizeof(buffer_view_object));

	zend_object_std_init(&intern->std, class_type TSRMLS_CC);
	object_properties_init(&intern->std, class_type);

	{
		zend_class_entry *base_class_type = class_type;
		while (base_class_type->parent) {
			base_class_type = base_class_type->parent;
		}

		if (base_class_type == int8_array_ce) {
			intern->type = buffer_view_int8;
		} else if (base_class_type == uint8_array_ce) {
			intern->type = buffer_view_uint8;
		} else if (base_class_type == int16_array_ce) {
			intern->type = buffer_view_int16;
		} else if (base_class_type == uint16_array_ce) {
			intern->type = buffer_view_uint16;
		} else if (base_class_type == int32_array_ce) {
			intern->type = buffer_view_int32;
		} else if (base_class_type == uint32_array_ce) {
			intern->type = buffer_view_uint32;
		} else if (base_class_type == float_array_ce) {
			intern->type = buffer_view_float;
		} else if (base_class_type == double_array_ce) {
			intern->type = buffer_view_double;
		} else {
			zend_error(E_ERROR, "Buffer view does not have a valid base class");
		}
	}

	retval.handle = zend_objects_store_put(
			intern,
			(zend_objects_store_dtor_t) zend_objects_destroy_object,
			(zend_objects_free_object_storage_t) linger_array_buffer_view_free_object_storage,
			NULL
			TSRMLS_CC
			);

	retval.handlers = &linger_array_buffer_view_handlers;
	return retval;
}
```

但是上述代码还需要特别说明的是，在`create_object`handler中我们额外的使用迭代来获取`buffer view`的基类，这是很有必要的，
因为很视图类有可能会是一个派生类。

然后就是`clone`handler的实现：

```c
static zend_object_value linger_array_buffer_view_clone(zval *object TSRMLS_DC)
{
	buffer_view_object *old_object = zend_object_store_get_object(object TSRMLS_CC);
	zend_object_value new_object_val = linger_array_buffer_view_create_object(Z_OBJCE_P(object) TSRMLS_CC);
	buffer_view_object *new_object = zend_object_store_get_object_by_handle(new_object_val.handle TSRMLS_CC);

	zend_objects_clone_members(&new_object->std, new_object_val, &old_object->std, Z_OBJ_HANDLE_P(object) TSRMLS_CC);

	new_object->buffer_zval = old_object->buffer_zval;
	if (new_object->buffer_zval) {
		Z_ADDREF_P(new_object->buffer_zval);
	}

	new_object->buf.as_int8 = old_object->buf.as_int8;
	new_object->offset = old_object->offset;
	new_object->length = old_object->length;
	new_object->type = old_object->type;

	return new_object_val;
}
```

接下来我们来实现视图类的构造函数，代码平淡无奇：

```c
PHP_FUNCTION(linger_array_buffer_view_ctor)
{
	zval *buffer_zval;
	long offset = 0, length = 0;
	buffer_object *buffer_intern;
	buffer_view_object *view_intern;
	zend_error_handling error_handling;

	zend_replace_error_handling(EH_THROW, NULL, &error_handling TSRMLS_CC);
	if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "o|ll", &buffer_zval, linger_array_buffer_ce, &offset, &length) == FAILURE) {
		zend_restore_error_handling(&error_handling TSRMLS_CC);
		return;
	}

	zend_restore_error_handling(&error_handling TSRMLS_CC);

	view_intern = zend_object_store_get_object(getThis() TSRMLS_CC);
	buffer_intern = zend_object_store_get_object(buffer_zval TSRMLS_CC);

	if (offset < 0) {
		zend_throw_exception(NULL, "Offset must be non-negative", 0 TSRMLS_CC);
		return;
	}
	if (offset >= buffer_intern->length) {
		zend_throw_exception(NULL, "Offset has to be smaller than the buffer length", 0 TSRMLS_CC);
		return;
	}
	if (length < 0) {
		zend_throw_excpetion(NULL, "Length must be positive or zero", 0 TSRMLS_CC);
		return;
	}
	view_intern->offset = offset;
	view_intern->buffer_zval = buffer_zval;
	Z_ADDREF_P(buffer_zval);

	{
		size_t bytes_per_element = linger_buffer_view_get_bytes_per_element(view_intern);
		size_t max_length = (buffer_intern->length - offset) / bytes_per_element;
		if (length == 0) {
			view_intern->length = max_length;
		} else if (length > max_length) {
			zend_throw_exception(NULL, "Length is large than the buffer", 0 TSRMLS_CC);
			return;
		} else {
			view_intern->length = length;
		}
	}
	view_intern->buf.as_int8 = buffer_intern->buffer;
	view_intern->buf.as_int8 += offset;
}
```

构造函数中使用到了一个`buffer_view_get_bytes_per_element`，它是用来返回每种视图类型的基本类型占用多少字节的空间：

```c
size_t linger_buffer_view_get_bytes_per_element(buffer_view_object *intern)
{
	switch (intern->type) {
		case buffer_view_int8:
		case buffer_view_uint8:
			return 1;
		case buffer_view_int16:
		case buffer_view_uint16:
			return 2;
		case buffer_view_int32:
		case buffer_view_uint32:
		case buffer_view_float:
			return 4;
		case buffer_view_double:
			return 8;
		default:
			zend_error_noreturn(E_ERROR, "Invalid buffer view type");
			
	}	
}
```

在这里，需要特别说明的是，`ArrayBuffer`和`ArrayBufferView`的联系是通过`union{}buf`这个联合体存储，这里的用法是非常巧妙的，
由于联合体的特性，它在一个共有的空间内每次至多只存储一种数据类型，而其中存放的又是某种类型的指针（其实就是一个数组）,在构
造视图类的时候，我们将视图结构体中的buf的起始位置指向`ArrayBuffer`中的`buffer`的起始位置，然后在进行插入和取出的时候，充
分利用c语言数组和指针的特性，很巧妙的对内存区块进行读写，从而达到目的。

到此，一切准备工作已经完成了，我们可以开始写真正的视图操作函数。

首先实现`offset set`操作:

```c
void linger_buffer_view_offset_set(buffer_view_object *intern, long offset, zval *value)
{
	if (intern->type == buffer_view_float || intern->type == buffer_view_double) {
		Z_ADDREF_P(value);
		convert_to_double_ex(&value);
		if (intern->type == buffer_view_float) {
			intern->buf.as_float[offset] = Z_DAVL_P(value);
		} else {
			intern->buf.as_double[offset] = Z_DAVL_P(value);
		}
		zval_ptr_dtor(&value);
	} else {
		Z_ADDREF_P(value);
		convert_to_long_ex(&value);
		switch (intern->type) {
			case buffer_view_int8:
				intern->buf.as_int8[offset] = Z_LVAL_P(value);
				break;
			case buffer_view_uint8:
				intern->buf.as_uint8[offset] = Z_LVAL_P(value);
				break;
			case buffer_view_int16:
				intern->buf.as_int16[offset] = Z_LVAL_P(value);
				break;
			case buffer_view_uint16:
				intern->buf.as_uint16[offset] = Z_LVAL_P(value);
				break;
			case buffer_view_int32:
				intern->buf.as_int32[offset] = Z_LVAL_P(value);
				break;
			case buffer_view_uint32:
				intern->buf.as_uint32[offset] = Z_LVAL_P(value);
				break;
			default:
				zend_error(E_ERROR, "Invalid buffer view type");
		}	
		zval_ptr_dtor(&value);
	}
}

PHP_FUNCTION(linger_array_buffer_view_offset_set)
{
	buffer_view_object *intern;
	long offset;
	zval *value;
	if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "lz", &offset, &value) == FAILURE) {
		return;
	}
	intern = zend_object_store_get_object(getThis() TSRMLS_CC);
	if (offset < 0 || offset >= intern->length) {
		zend_throw_excpetion(NULL, "Offset is outside the buffer range", 0 TSRMLS_CC);
		return;
	}
	linger_buffer_view_offset_set(intern, offset, value);

	RETURN_TRUE;
}
```

代码平淡无奇，这里就不做过多的解释，同样很简单的来实现`offset get`操作：

```c
zval *linger_buffer_view_offset_get(buffer_view_object *intern, size_t offset)
{
	zval *retval;
	MAKE_STD_ZVAL(retval);

	switch (intern->type) {
		case buffer_view_int8:
			ZVAL_LONG(retval, intern->buf.as_int8[offset]);
			break;
		case buffer_view_uint8:
			ZVAL_LONG(retval, intern->buf.as_uint8[offset]);
			break;
		case buffer_view_int16:
			ZVAL_LONG(retval, intern->buf.as_int16[offset]);
			break;
		case buffer_view_uint16:
			ZVAL_LONG(retval, intern->buf.as_uint16[offset]);
			break;
		case buffer_view_int32:
			ZVAL_LONG(retval, intern->buf.as_int32[offset]);
			break;
		case buffer_view_uint32: {
			uint32_t value = intern->buf.as_uint32[offset];
			if (value <= LONG_MAX) {
				ZVAL_LONG(retval, value);
			} else {
				ZVAL_DOUBLE(retval, value);
			}
			break;
		}
		case buffer_view_float:
			ZVAL_DOUBLE(retval, intern->buf.as_float[offset]);
			break;
		case buffer_view_double:
			ZVAL_DOUBLE(retval, intern->buf.as_double[offset]);
			break;
		default:
			zend_error_noreturn(E_ERROR, "Invalid buffer view type");
	}
	return retval;
}

PHP_FUNCTION(linger_array_buffer_view_offset_get)
{
	buffer_view_object *intern;
	long offset;
	if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "l", &offset) == FAILURE) {
		return;
	}

	intern = zend_object_store_get_object(getThis() TSRMLS_CC);
	if (offset < 0 || offset >= intern->length) {
		zend_throw_exception(NULL, "Offset is outside the buffer range", 0 TSRMLS_CC);
		return;
	}

	zval *retval;
	retval = linger_buffer_view_offset_get(intern, offset);
	RETURN_ZVAL(retval, 1, 1);
}
```

对于剩下的`offsetExists`和`offsetUnset`这里就不再赘述了。而且这里只实现了数据的基本存取操作，并没有实现`Iterator`接口，不支持`foreach`操作，
不过会在后续的文章中实现，今天的完整代码可到github上查看：[https://github.com/iliubang/php-ArrayBuffer.git](https://github.com/iliubang/php-ArrayBuffer.git)

编译后写个测试脚本：

```php
<?php

$buffer = new linger\ArrayBuffer(256);
var_dump($buffer);

$int32 = new linger\ArrayBufferView\Int32Array($buffer);
var_dump($int32);
$uint8 = new linger\ArrayBufferView\UInt8Array($buffer);
var_dump($uint8);

for ($i = 0; $i < 255; $i++) {
    $uint8[$i] = $i;
}

for ($i = 0; $i < 255; $i++) {
    echo $int32[$i], "\n";
}

```

同样的代码可以用js测试一下：

```javascript
var buffer = new ArrayBuffer(256);
var uint8 = new Uint8Array(buffer);
var int32 = new Int32Array(buffer);
for (i = 0; i < 255; i++) {uint8[i] = i;}
for (i = 0; i < 255; i++) {console.log(int32[i]);}
```

结果是一致的。
