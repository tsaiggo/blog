---
title: redis 6.0 学习笔记 -- 字符串
date: 2020-06-29 13:46:12
category: ["redis"]

---

简单动态字符串（Simple Dynamic Strings, SDS）是Redis的基本数据结构之一，用于存储字符串和整型数据。

SDS兼容C语言标准字符串处理函数，且在此基础上保证了二进制安全。
<!--more-->

### 首先，我们来开门见山的了解一下 SDS 的结构

```c
/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
/** 包括 sdshdr16、sdshdr32、sdshdr64 这三种类型，都只是将 sdshdr8 的 len 和 alloc 替换成不同长度的数据类型，其结构与 sdshdr8 保持一致 **/
```

上述的是 redis 3.2 版本之后的结构。

我们可以通过对比以前的版本，比如 redis 3.0 ，来思考为什么需要如此看似复杂，却又巧妙的数据结构。

```c
struct sdshdr {
    unsigned int len;
    unsigned int free;
    char buf[];
};
```

其实早在早期设计的时候，就已经存在着以下的几个优点（参考了 Redis 5 设计与源码分析 一书）：

1. 有单独的 len 和 free，可以快速的算出长度和可用空间。（联想到 java nio 中的 Buffer，其中的 capacity 和 limit 估计也是类似的设计思路）
2. 内容存放在 buf 中，SDS 对上层暴露的指针不是指向结构体 SDS 的指针，而是直接指向 buf 的指针。上层可像读取 C 字符串一样读取 SDS 的内容，兼容 C 语言处理字符串的各种函数。
3. 由于有长度统计变量 len 的存在，读写字符串时不依赖“\0”终止符，保证了二进制安全。

但是，我们需要考虑到另一个问题，不同长度的字符串是否有必要占用相同大小的头部？
所以，在此问题上，redis 3.2 做出了改进，出现了针对不同长度字符串时的几个结构体，如 sdshdr5、sdshdr8、sdshdr16、sdshdr32、sdshdr64。可以观察到，不论 sds 选择了哪种结构体作为底层实现，都存在一个标记符 flags。

flags 的数据类型是 char，说明是八位长度，后五位在 sdshdr5 中作为长度标记，在其他结构中是未使用的，而前三位，则是统一的 type 标示位。类似的奇技淫巧在 java 的 ConcurrentHashMap 中的 sizeCtl 也有实现。

```c
#define SDS_TYPE_5  0		//二进制位 000
#define SDS_TYPE_8  1		//二进制位 001
#define SDS_TYPE_16 2		//二进制位 010
#define SDS_TYPE_32 3		//二进制位 011
#define SDS_TYPE_64 4		//二进制位 100
```

这其实是 redis 想要将内存利用做到极限的体现，包括对比两个版本之间结构体的差异，我们可以看到 \_\_attribute\_\_ ((\_\_packed\_\_)) 这样子的关键字，这是使得结构体内存对齐的修饰。

### 然后，让我们来看一下 SDS 的几个常见方法

#### sds sdsnewlen(const void *init, size_t initlen)

```c
/* Create a new sds string with the content specified by the 'init' pointer
 * and 'initlen'.
 * If NULL is used for 'init' the string is initialized with zero bytes.
 * If SDS_NOINIT is used, the buffer is left uninitialized;
 *
 * The string is always null-termined (all the sds strings are, always) so
 * even if you create an sds string with:
 *
 * mystring = sdsnewlen("abc",3);
 *
 * You can print the string with printf() as there is an implicit \0 at the
 * end of the string. However the string is binary safe and can contain
 * \0 characters in the middle, as the length is stored in the sds header. */
sds sdsnewlen(const void *init, size_t initlen) {
    void *sh;
    sds s;
    char type = sdsReqType(initlen);
    /* Empty strings are usually created in order to append. Use type 8
     * since type 5 is not good at this. */
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
    int hdrlen = sdsHdrSize(type);
    unsigned char *fp; /* flags pointer. */

    sh = s_malloc(hdrlen+initlen+1);
    if (sh == NULL) return NULL;
    if (init==SDS_NOINIT)
        init = NULL;
    else if (!init)
        memset(sh, 0, hdrlen+initlen+1);
    s = (char*)sh+hdrlen;
    fp = ((unsigned char*)s)-1;
    switch(type) {
        case SDS_TYPE_5: {
            *fp = type | (initlen << SDS_TYPE_BITS);
            break;
        }
        // 其实下文都是重复代码，
        case (SDS_TYPE_8 or SDS_TYPE_16 or SDS_TYPE_32 or SDS_TYPE_64): {
            SDS_HDR_VAR(8 or 16 or 32 or 64,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
    }
    if (initlen && init)
        memcpy(s, init, initlen);
    s[initlen] = '\0';
    return s;
}
```

其实这只是一个比较通俗的写法，值得注意的是：

1. ``` if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;``` 源码中存在着这样的一句话，注释中也是这样子写到```Empty strings are usually created in order to append. Use type 8 since type 5 is not good at this.``` 其实这是因为 type_5 的结构和 type_8 及其他的结构大不相同，原因可能是创建空字符串后，其内容可能会频繁更新而引发扩容，故创建时直接创建为type_8。
2. ```memset(sh, 0, hdrlen+initlen+1);``` 实际长度是需要 +1 的，因为在字符串结尾，需要有一个"\0"终止符。
3. ```typedef char *sds;``` 实际上返回值是指向sds结构buf字段的指针。这样子设计的好处是直接对上层提供了字符串内容指针，兼容了部分C函数，且通过偏移能迅速定位到SDS结构体的各处成员变量。

#### 释放一个字符串

```c
/* Free an sds string. No operation is performed if 's' is NULL. */
#define s_free free
void sdsfree(sds s) {
    if (s == NULL) return;
    s_free((char*)s-sdsHdrSize(s[-1]));
}

/* Modify an sds string in-place to make it empty (zero length).
 * However all the existing buffer is not discarded but set as free space
 * so that next append operations will not require allocations up to the
 * number of bytes previously available. */
void sdsclear(sds s) {
    sdssetlen(s, 0);
    s[0] = '\0';
}
```

1. ```sdsfree``` 是直接释放内存，```s[-1]```这个操作会经常见到，上文中也有提到，redis在字符串操作时，返回的都是直接指向buf结构的指针，可以回看上文的 sds 结构，s[-1] 表示为 flags 字段。
2. ```sdsclear```是假释放内存，这是性能优化的手段，同样的，在 java 的 nio 实现中也有着类似的手段。这里不会进行任何内存释放的操作，而且重置```uint8_t len; /* used */```，在后续的使用中进行值覆盖的操作。