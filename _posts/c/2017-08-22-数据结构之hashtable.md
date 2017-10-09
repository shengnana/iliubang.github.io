---
layout: post
title: 数据结构之hashtable
subtitle: 给php再扩展一个hashtable
author: 刘邦
weather: sunny
catalog: true
tags: [c, algorithm]
category: c
use_math: true
---

## hashtable

哈希表又叫散列表，是实现字典操作的中有效数据结构。通常来说，一个hash table包含了一个数据，其中的数据通过index来访问。
而hash table的基本原理就是通过hash函数建立起所有可能的index与其对应的位置的联系。一个hash函数接收一个key，返回其hash code，
key的类型是可变的，而hash code 是一个整型。

由于计算一个hash值和通过index访问一个数据都是常量级的时间复杂度，所以我们可以通过这中特性实现常量级时间复杂度的查找。
如果一个hash函数能够保证不会有两个不同的key生成相同的hash值，那么这样的hash table就被称为是直接定址。然而，这只是一想法而已，
实际上这种hash table在现实中却是不常用的。

### Chained Hash Table

链式hash表从本质上来讲，就是一个存放了一组链表的数组。每个链表可以看做是一个槽，我们把元素通过hash函数找到一个hash值，然后把元素的值
放入到数组中与改hash值对应的槽中。

![chained hashtable](/img/2017-08-23/chained_hashtable.png)

#### Collision Resolution

当有两个key被hash到了同一个位置，就会产生冲突。链式hash表有一种简单的冲突解决办法：当冲突产生时，元素被简单的放在同一个槽里。这样做可能带来的问题就是，
如果在同一个位置上出现很多冲突，这个槽就会变得越来越长，这样当我们访问这个槽中的元素的时候，所花的时间也就会越来越长。

在理想状态下，我们希望所有的槽能够以同样的速度增长，这样他们就可以尽可能地保持小的容量和相同的大小。换句话说，我们的目标就是尽可能均匀分散所有的元素。这种情况
在理论上称为均匀散列。而实际中，我们只能尽可能近似的到达这种状态。

如果插入表中的元素数量远大于表中槽的数量，那么即使在均匀分布的情况下，表中的槽也会变得越来越深，表的性能也会随之迅速降低。因此我们必须要特别注意一个hash表的负载因子，
其定义为：

$$
\alpha = n / m
$$

其中$n$是表中元素的个数，$m$是槽的数量。链式hash表的负载因子表示在均匀散列的情况下，表中每一个槽的最大期望存放的元素个数。
例如：有一个hash表，$m = 1699$, $n = 3198$，那么这个hash表的负载因子为$\alpha = 3198/1699 = 2$。因此在这种情况下，我们说
该hash表中每一个槽中的元素数量最大不要超过2。

#### 选择hash函数

一个好的hash函数的目标是尽可能近似于均匀散列。定义一个hash函数$f$，它将键$k$映射到hash表中的位置$x$，$x$称为$k$的hash code：

$$
h(k) = x
$$

##### Division Method

有一个整数键$k$，有一种最简单的将$k$映射到hash表中槽位的方法就是取余数:

$$
h(h) = k\,mod\,m
$$

##### Multiplication Method

与求余法不同的一种方法是乘法，它将整型键$k$乘以一个常数$A (0 < A < 1)$，取所乘结果的小数部分，然后再乘以$m$，取结果的整数部分。通常情况下，$A$取$0.618$，
它由$\sqrt{5}$减$1$再除以$2$得到：

$$
h(k) = \lfloor m (kA\,mod\,1) \rfloor，其中 A \approx (\sqrt{5} - 1) / 2
$$

#### 链式hash table的实现

基于以上理论，实现一个hash表已经显得非常简单了。下面直接给出代码：

```c
#include <stdlib.h>
#include <stdio.h>
#include <limits.h>
#include <string.h>

#define linger_free(ptr)   if (ptr) free(ptr)

typedef struct entry_s {
    char *key;
    char *value;
    struct entry_s *next;
} entry_t;

typedef struct hashtable_s {
    int size;
    entry_t **table;
} hashtable_t;

hashtable_t *ht_create(int size)
{
    hashtable_t *hashtable = NULL;
    int i;

    if (size < 1) {
        return NULL;
    }

    if ((hashtable = malloc(sizeof(hashtable_t))) == NULL) {
        return NULL;
    }
    
    if ((hashtable->table = malloc(sizeof(entry_t *) * size)) == NULL) {
        linger_free(hashtable);
        return NULL;
    }

    for (i = 0; i < size; i++) {
        hashtable->table[i] = NULL;
    }
    
    hashtable->size = size;
    return hashtable;
}

int ht_hash(hashtable_t *hashtable, char *key)
{
    const char *ptr;
    unsigned int val;
    val = 0;
    ptr = key;
    while (*ptr != '\0') {
        unsigned int tmp;
        val = (val << 4) + (*ptr);
        if (tmp = (val & 0xf0000000)) {
            val = val ^ (tmp >> 24);
            val = val ^ tmp;
        }
        ptr++;
    }
    return val % hashtable->size;
}

entry_t *ht_newpair(char *key, char *value)
{
    entry_t *newpair;
    if ((newpair = malloc(sizeof(entry_t))) == NULL) {
        return NULL;
    }

    if ((newpair->key = strdup(key)) == NULL) {
        return NULL;
    }
    
    if ((newpair->value = strdup(value)) == NULL) {
        return NULL;
    }
    newpair->next = NULL;

    return newpair;
}

void ht_set(hashtable_t *hashtable, char *key, char *value)
{
    int bin = 0;
    entry_t *newpair = NULL;
    entry_t *next = NULL;
    entry_t *last = NULL;

    bin = ht_hash(hashtable, key);

    next = hashtable->table[bin];

    while (next != NULL && next->key != NULL && strcmp(key, next->key) > 0) {
        last = next;
        next = next->next;
    }

    if (next != NULL && next->key != NULL && strcmp(key, next->key) == 0) {
        linger_free(next->value);
        next->value = strdup(value);
    } else {
        newpair = ht_newpair(key, value);
        if (next == hashtable->table[bin]) {
            newpair->next = next;
            hashtable->table[bin] = newpair;
        } else if (next == NULL) {
            last->next = newpair;
        } else {
            newpair->next = next;
            last->next = newpair;
        }
    }
}

char *ht_get(hashtable_t *hashtable, char *key)
{
    int bin = 0;
    entry_t *pair;
    bin = ht_hash(hashtable, key);
    pair = hashtable->table[bin];
    while (pair != NULL && pair->key != NULL && strcmp(key, pair->key) > 0) {
        pair = pair->next;
    }

    if (pair == NULL || pair->key == NULL || strcmp(key, pair->key) != 0) {
        return NULL;
    }

    return pair->value;
}

void ht_destroy(hashtable_t *hashtable) 
{
    entry_t *curr, *next;
    if (hashtable->size > 0) {
        for (int i = 0; i < hashtable->size; i++) {
            if (hashtable->table[i] == NULL) {
                continue;
            }             
            curr = hashtable->table[i];
            while (curr != NULL) {
                next = curr->next; 
                linger_free(curr->key);
                linger_free(curr->value);
                linger_free(curr);
                curr = next;
            }
             
        } 
    }
    linger_free(hashtable);
}
```

至此我们就实现了一个简单的链式hash table。

### 给php扩展一个hash table
接下来我们把这个hash table写成php扩展。由于太简单，这里就不再详细讲述实现过程，源代码可以访问[https://github.com/iliubang/php_hashtable_extension.git](https://github.com/iliubang/php_hashtable_extension.git)

写了个脚本测试下性能:

```php
<?php
$ht = new linger\Hashtable(65535);

$n = 10000;

$start = microtime(true);
for ($i = 0; $i < $n; $i++) {
    $ht->set("hello{$i}", "world{$i}");
}

for ($i = 0; $i < $n; $i++) {
    $ht->get("hello{$i}");
}

echo "HashTable:" . (microtime(true) - $start), PHP_EOL;

$arr = [];

$start = microtime(true);
for ($i = 0; $i < $n; $i++) {
   $arr["hello{$i}"] = "world{$i}"; 
}

for ($i = 0; $i < $n; $i++) {
    $tmp = $arr["hello{$i}"];
}
echo "Array:" . (microtime(true) - $start), PHP_EOL;
```

测试结果还是很不错的：

```shell
liubang@venux:~/workspace/c/php-hashtable-extension/tests$ php test2.php 
HashTable:0.009289026260376
Array:0.0061149597167969
```

