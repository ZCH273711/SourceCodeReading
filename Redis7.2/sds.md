# 一、Redis7.2 源码阅读笔记 —— Simple Dynamic String（sds）

sds的数据结构源码主要位于src目录下的sds.h和sds.c两个文件。其中sds.h文件主要是结构体的定义以及一些结构体成员访问方法，所有该类方法都声明为内联函数。

## sds结构体的定义
总的来说，redis一共有5中sds结构体定义，分别使用不同长度的int类型来记录字符串的长度，主要是为了效率和空间利用的目的。源码如下。
```C
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
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

除了sdshdr5类型，其他四种sds类型都有len和alloc两个字段，表示字符串的长度和分配的可用长度。sdshdr5类型的长度只是用5位bit位来记录，位于flags的高5位。flags的低3位用来记录sds的类型，即是上面五种sdshdr的哪一种，在sds.h文件中通过宏定义定义类型。结构体的最后一个成员是一个柔性数组，相比指针可以减少一次内存分配，并且在连续内存有空闲的情况下可以通过realloc高效扩容。类型宏定义如下：

```c
#define SDS_TYPE_5  0
#define SDS_TYPE_8  1
#define SDS_TYPE_16 2
#define SDS_TYPE_32 3
#define SDS_TYPE_64 4
#define SDS_TYPE_MASK 7
```

在做类型比较是，只需要将flag和SDS_TYPE_MASK做按位与操作即可得到sds具体类型。

另外需要注意的是，创建sds结构体过后，返回给上层应用的是buf的指针，即上层应用看到的就是一个char类型的数组。但是，因为结构体的定义让编译器关闭了字节对齐，所以buf到flags的偏移位-1，可以直接通过buf指针访问flags。再通过flags判断sds具体结构体类型后可以获取len、alloc等信息。

## sds.h中的函数定义

该文件中主要声明并定义了如下函数，其中sds的定义为typedef char *sds，即指向char类型的指针。所有函数都会通过s-1访问flags判断结构类型，然后做出相应的操作。

```C
static inline size_t sdslen(const sds s); // 获取sds的长度
static inline size_t sdsavail(const sds s);	// 获取sds可用的空间
static inline void sdssetlen(sds s, size_t newlen); // 设置sds新的长度
static inline void sdsinclen(sds s, size_t inc); // 将sds长度增加一个值
static inline size_t sdsalloc(const sds s); // 获取sds最大可用长度
static inline void sdssetalloc(sds s, size_t newlen); // 设置sds最大可用长度
```

## sds.h中的函数声明

除了上述声明并定义了的函数，该文件还声明了其他许多对sds进行操作的函数，基本都在sds.c文件中实现了。例如以下函数声明：

```c
sds sdsnewlen(const void *init, size_t initlen);
sds sdstrynewlen(const void *init, size_t initlen);
sds sdsnew(const char *init);
sds sdsempty(void);
sds sdsdup(const sds s);
void sdsfree(sds s);
sds sdsgrowzero(sds s, size_t len);
sds sdscatlen(sds s, const void *t, size_t len);
sds sdscat(sds s, const char *t);
sds sdscatsds(sds s, const sds t);
sds sdscpylen(sds s, const char *t, size_t len);
sds sdscpy(sds s, const char *t);
```

## sds创建

sds创建主要涉及几个函数，这些函数都在sds.c中实现。

```c
sds sdsempty(void)	// 创建长度为0的字符串
sds sdsnew(const char *init) // 从一个C字符串创建
sds sdsdup(const sds s)	// 复制一个字符串
sds sdsnewlen(const void *init, size_t initlen)
sds sdstrynewlen(const void *init, size_t initlen)
sds _sdsnewlen(const void *init, size_t initlen, int trymalloc)
```

前三个函数都会调用第四个函数，最后一个函数在sds数据结构实现时没有使用，主要是其他模块使用。第四五个函数都会调用第六个函数，所以_sdsnewlen函数才是创建sds的逻辑。下面截取该函数前部分代码进行注释说明。

```c
sds _sdsnewlen(const void *init, size_t initlen, int trymalloc) {
    void *sh;
    sds s;
    char type = sdsReqType(initlen);    // 根据长度，选择合适的类型
    // sdshdr5已经弃用，改用sdshdr8，因为字符串会频繁append操作，sdshdr5分配的空间太少
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;  
    int hdrlen = sdsHdrSize(type);  // 计算结构体长度
    unsigned char *fp; /* flag的指针 */
    size_t usable;

    assert(initlen + hdrlen + 1 > initlen); /* 防止长度溢出 */
    sh = trymalloc?
        s_trymalloc_usable(hdrlen+initlen+1, &usable) :
        s_malloc_usable(hdrlen+initlen+1, &usable); // 分配内存，usable是实际分配的内存大小
    if (sh == NULL) return NULL;
    if (init==SDS_NOINIT)
        init = NULL;
    else if (!init)
        memset(sh, 0, hdrlen+initlen+1);
    s = (char*)sh+hdrlen;
    fp = ((unsigned char*)s)-1;
    ......
}
```

## sds字符串拼接

拼接涉及以下几个函数：

```c
sds sdscatlen(sds s, const void *t, size_t len) // 被下面两个函数调用，拼接的具体逻辑
sds sdscat(sds s, const char *t) // 拼接一个sds和一个C字符串
sds sdscatsds(sds s, const sds t)  // 拼接两个sds
sds _sdsMakeRoomFor(sds s, size_t addlen, int greedy) // 扩容逻辑
```

具体的关系如注释所描述的，sdscatlen会调用_sdsMakeRoomFor进行扩容。sdscatlen的代码如下，首相调用 _sdsMakeRoomFor函数对s进行扩容，然后直接使用memcpy函数将t中的内容复制到s后面，最后设置s长度，并在sds字符串最后加上结束标志位。

```c
size_t curlen = sdslen(s);

s = sdsMakeRoomFor(s,len);
if (s == NULL) return NULL;
memcpy(s+curlen, t, len);
sdssetlen(s, curlen+len);
s[curlen+len] = '\0';
return s;
```

## sds字符串扩容

如上面所说，扩容的具体逻辑体现在_sdsMakeRoomFor函数中。由于具体代码比较长，所以这里摘取关键点说明。

1. 如果可用的空间大于需要新增的字符串长度，则直接返回。

```c
void *sh, *newsh;
size_t avail = sdsavail(s);
size_t len, newlen, reqlen;
char type, oldtype = s[-1] & SDS_TYPE_MASK;
int hdrlen;
size_t usable;

/* Return ASAP if there is enough space left. */
if (avail >= addlen) return s;
```

2. 根据greedy标志位，扩容不同的容量，如果标志位为1，则扩容比需要的空间更大的空间，否则扩容到刚好的空间。

```c
/* 下面的if语句，根据newlen的大小，决定怎样扩容 
当newlen小于SDS_MAX_PREALLOC时，扩容两倍，否则扩容一个SDS_MAX_PREALLOC
其中SDS_MAX_PREALLOC=2**20即1M */
if (greedy == 1) {
	if (newlen < SDS_MAX_PREALLOC)
		newlen *= 2;
	else
		newlen += SDS_MAX_PREALLOC;
}
```

3. 根据计算出的扩容空间，判断具体的sds类型，如果和原类型相同，则尝试在当前内存位置后面申请连续内存，否则重新在内存区域分配结构体头部和字符串空间。

```C
if (oldtype==type) {
    newsh = s_realloc_usable(sh, hdrlen+newlen+1, &usable); // 新的sds结构体指针，底层调用realloc函数
    if (newsh == NULL) return NULL;
    s = (char*)newsh+hdrlen;
} else {
    newsh = s_malloc_usable(hdrlen+newlen+1, &usable); // 底层调用malloc函数
    if (newsh == NULL) return NULL;
    memcpy((char*)newsh+hdrlen, s, len+1);
    s_free(sh);
    s = (char*)newsh+hdrlen;
    s[-1] = type;
    sdssetlen(s, len);
}
```

## 其他功能

除了上面提到的创建、拼接sds，redis低层的sds还提供了例如copy、join、trim、toupper、tolower、substr以及和long long类型转换等函数方便上层应用调用。







