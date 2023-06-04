# Redis7.2源码阅读笔记——ziplist

压缩列表（ziplist）是redis中zset的两种底层实现之一，另一种是skiplist，在另一篇笔记中已经对skiplist主要部分分析了。这篇笔记用来记录阅读redis中的ziplist实现源码的笔记。

## ziplist简介

ziplist的实现主要在ziplist.h和ziplist.c文件中。在ziplist.c的源码的官方注释中可以了解到，ziplist是一种被设计来提高内存使用率的数据结构，他可以存储字符串和整数类型，在ziplist头尾压入和弹出元素的时间复杂度为O(1)，但是考虑到在中间位置插入和删除会涉及内存的重新分配和复制，所以整体的时间复杂度收到ziplist占用空间的影响。

压缩列表展开的形式如下：

<zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>

注意：压缩列表的四个特殊字段都按照little endian编码，即低位编址（操作系统中有讲）。

- 其中zlbytes是一个32位无符号类型整数，用来记录整个压缩列表占用的字节数（包括zlbytes自己的4字节）。设置该成员是为了改变压缩列表大小时不用去遍历整个列表。
- zltail元素也是一个32位无符号整数类型，用来记录列表中最后一个元素的字节偏移
- zllen元素是一个16位无符号整数类型，用来记录整个列表的元素个数，如果元素个数大于2^16 - 2，则设置为2^16 - 1，并且需要遍历整个列表才能知道有多少个元素。
- zlend是一个标记字节，恒为0xFF，标志整个压缩列表的结束。

## ziplist中的entry

压缩列表中的每一个entry元素展开如下：

<prevlen> <encoding> <entry-data>

- 其中prevlen表示前一个元素所占字节数，方便反向遍历。该字段的编码规则如下：
  - 如果前一个数据元素总长度小于254，则只需要一个字节。（为什么不是255呢，因为压缩列表中的zlend就是255，标志列表的结束，所以不能为255）
  - 否则当前一个元素的总长度大于等于254，则需要5个字节，其中第一个字节恒为0xFE(254)，实际表示前一个元素长度为后四个字节。
- encoding表示当前元素的类型，有字符串和整数两种，同时encoding也会含有字符串的长度信息。可以根据encoding的第一字节的高二位分别不同长度的字符串和整数，具体的编码规则如下：
  - |00pppppp| 长度不大于63的字符串，encoding为一字节，后六位记录字符串的长度。
  - |01pppppp|qqqqqqqq| 2字节，字节长度不大于16383的字符串，后14位记录具体长度。注意：后14位长度采用大端地址（Big Endian）。
  - |10000000|qqqqqqqq|rrrrrrrr|ssssssss|tttttttt| 5字节，长度大于等于16384的字符串，第一字节的后六位没有使用，使用后面四个字节记录具体长度，同样采用大端地址。
  - |11000000| 3字节，16位有符号整数，其中一字节encoding，接着后面两字节<entry-data>的整数。
  - |11010000| 5字节，32位有符号整数，同样后面接4字节整数。
  - |11100000| 9字节，64位有符号整数，后面接8个字节。
  - |11110000| 4字节，24位有符号整数。
  - |11111110| 2字节，后面接8位有符号整数。
  - |1111xxxx| 12个立即数，也就是说后面四位直接表示数字，从0001到1101（0000，1110，1111不可用），一共12个数，从0开始。
  - 注意：和特殊字段一样，整数采用小端地址存储。
- entry-data即存储的数据，如果存储的数据为小整数类型，则这部分会省略，直接并入encoding字段。

## 压缩列表的创建

压缩列表的创建的逻辑在unsigned char *ziplistNew(void) 函数中实现，声明和定义分别位于ziplist.h，ziplist.c。具体代码如下，首先计算压缩列表头尾的字节长度，然后在内存中进行分配，随后对zlbytes、zltail、zllen三个字段进行赋值。其中intrev32ifbe函数调用了endiancov.c文件中的intrev32函数，该函数是为了将32位整数的小端存储转换为大端存储。

```c
unsigned char *ziplistNew(void) {
    unsigned int bytes = ZIPLIST_HEADER_SIZE+ZIPLIST_END_SIZE;
    unsigned char *zl = zmalloc(bytes);
    ZIPLIST_BYTES(zl) = intrev32ifbe(bytes);
    ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(ZIPLIST_HEADER_SIZE);
    ZIPLIST_LENGTH(zl) = 0;
    zl[bytes-1] = ZIP_END;
    return zl;
}
```

## 压缩列表的插入

压缩列表的插入操作由unsigned char *ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen)函数实现，同样在ziplist.h中定义，在ziplist.c中实现。注意，该函数的调用需要知道在压缩列表中的插入位置（p）。该函数的实现很简单，直接调用另一个函数unsigned char *__ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen)，具体的逻辑在这个函数中实现，同样，由于有点复杂，这里分段分析。

**首先**是计算p指向的元素前一个元素的长度。可以分为三种情况：

- 如果当前列表为空，则直接插入，因为前一个元素长度为0.
- 如果插入位置在列表尾部，则需要将前一个元素的三个字段的空间加起来。
- 如果插入位置在列表中间，则只需要获取当前p指向元素所记录的prevlen字段即可。

代码如下，首先判断p指向的位置（需要插入的位置）是否为列表尾部（ZIP_END为255），如果不为尾部，则直接解码p指向的元素。否则通过ZIPLIST_ENTRY_TAIL宏定义获取最后一个元素，计算三个字段的和并返回。

```c
if (p[0] != ZIP_END) {
    ZIP_DECODE_PREVLEN(p, prevlensize, prevlen);
} else {
    unsigned char *ptail = ZIPLIST_ENTRY_TAIL(zl);
    if (ptail[0] != ZIP_END) {
        prevlen = zipRawEntryLengthSafe(zl, curlen, ptail);
    }
}
```

**第二步**，计算待插入元素所需要的字节空间，代码如下。首先尝试将传入的元素（无论字符串还是整数传入的都是字符串）进行编码，对应zipTryEncoding函数，该函数的实现调用了sds中实现的string2ll函数，尝试将字符串转换为整数。string2ll函数要求传入的字符串为严格的“整数格式”否则转换失败。

尝试将字符串转换为整数后，如果成功，则计算整数需要的存储字节数，否则数据作为字符串存储，数据所占空间为字符串长度。注意到目前为止reqlen只记录了数据在entry所需要的空间，所以后面会将prevlen元素需要的空间和encoding元素需要的空间加上得到插入元素entry总的长度。具体得到的结果可以参考上面对entry结构体的介绍，因为得到prevlen和encoding元素所需长度的两个函数的判断逻辑基本符合上面对entry的分析，当然，感兴趣还是可以去看这两个函数的源码详细了解。

```c
if (zipTryEncoding(s,slen,&value,&encoding)) {

    reqlen = zipIntSize(encoding);
} else {
    reqlen = slen;
}
reqlen += zipStorePrevEntryLength(NULL,prevlen);
reqlen += zipStoreEntryEncoding(NULL,encoding,slen);
```

**第三步**，计算下一个元素（还未在p处插入新元素时p指向的元素）所需空间的变化并分配空间，代码如下。在列表中见插入元素时，后面的第一个entry的prevlen字段可能会变化。因为插入过后它的前一个元素的长度已经改变了，并且，该字段的空间变化只有三种可能，减小4字节（此时插入的entry长度小于等于254字节）、不变、增大4字节（此时插入的entry长度大于254字节）。

当前面计算的reqlen < 4 && nextdiff == -4 时，会产生一个问题，就是ziplist的整体长度变小了，并且接着后面会调用ziplistResize函数（里面会调用realloc函数），则可能会被realloc函数回收多余的空间导致数据丢失，因为resize操作的时候还没有移动各个entry内容，只是缩短了原来ziplist的空间。所以，如果遇到这种情况则使用forcelarge标志，这样在后面resize操作时可以进行处理。

最后四行代码就是记录p相对于表头的offset，然后分配所需空间。

```c
int forcelarge = 0;
nextdiff = (p[0] != ZIP_END) ? zipPrevLenByteDiff(p,reqlen) : 0;
if (nextdiff == -4 && reqlen < 4) {
    nextdiff = 0;
    forcelarge = 1;
}

offset = p-zl;
newlen = curlen+reqlen+nextdiff;
zl = ziplistResize(zl,newlen);
p = zl+offset;
```

**第四步**，移动原先的内容，代码如下。移动后，修改下一个entry的prevlen字段，如果forcelarge为1，则该字段必为5个字节。否则会根据情况选择1字节或5字节。最后会根据情况修改ziplist中最后一个元素的偏移字段。

```c
if (p[0] != ZIP_END) {
    memmove(p+reqlen,p-nextdiff,curlen-offset-1+nextdiff);

    if (forcelarge)
        zipStorePrevEntryLengthLarge(p+reqlen,reqlen);
    else
        zipStorePrevEntryLength(p+reqlen,reqlen);

    ZIPLIST_TAIL_OFFSET(zl) =
        intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+reqlen);
        
    assert(zipEntrySafe(zl, newlen, p+reqlen, &tail, 1));
    if (p[reqlen+tail.headersize+tail.len] != ZIP_END) {
        ZIPLIST_TAIL_OFFSET(zl) =
            intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+nextdiff);
    }
} else {
    ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(p-zl);
}
```

**最后一步**，进行级联更新、正式插入数据并将列表长度加一，代码如下。对于级联更新这里可以举一个例子方便理解。即在当前插入的位置及其后面所有的entry的长度都为253，并且当前插入的元素导致原本长度为253字节的entry增加4个字节（prevlen），则后面所有的entry的prevlen字节空间都需要增加4字节。

```c
if (nextdiff != 0) {
    offset = p-zl;
    zl = __ziplistCascadeUpdate(zl,p+reqlen);
    p = zl+offset;
}

p += zipStorePrevEntryLength(p,prevlen);
p += zipStoreEntryEncoding(p,encoding,slen);
if (ZIP_IS_STR(encoding)) {
    memcpy(p,s,slen);
} else {
    zipSaveInteger(p,value,encoding);
}
ZIPLIST_INCR_LENGTH(zl,1);
return zl;
```

## 压缩列表的删除

列表的删除的函数为unsigned char *ziplistDelete(unsigned char *zl, unsigned char **p)，被声明在ziplist.h中，实现在ziplist.c中。该函数也是直接调用了一个内部函数unsigned char *__ziplistDelete(unsigned char *zl, unsigned char *p, unsigned int num)，后者函数可以从p指向的位置开始删除num个元素。

这里还是分段分析__ziplistDelete函数的代码。

**首先**是计算需要删除的字节数，代码如下。使用first记录p指向的第一个entry（也是第一个需要删除的entry）。然后向后遍历num个entry，为需要删除的entry个数。最后使用totlen计算总的需要删除的字节数。后面的步骤只有在totlen > 0的情况下才会进行，否则没有需要删除entry，直接返回。

```c
unsigned int i, totlen, deleted = 0;
size_t offset;
int nextdiff = 0;
zlentry first, tail;
size_t zlbytes = intrev32ifbe(ZIPLIST_BYTES(zl));

zipEntry(p, &first); 
for (i = 0; p[0] != ZIP_END && i < num; i++) {
    p += zipRawEntryLengthSafe(zl, zlbytes, p);
    deleted++;
}

assert(p >= first.p);
totlen = p-first.p; 
```

 **第二步**，经过第一步，此时p已经指向了需要被删除的entry后面的第一个entry。如果p指向的是列表尾部，则不需要修改什么；否则，p指向的是一个entry元素，需要修改该entry元素的prevlen元素，因为删除过后该entry前面的entry改变了。

如果p指向的是一个entry元素，首先调用zipPrevLenByteDiff函数计算prevlen字段的变化长度，同上面插入操作，nextdiff可能为-4，0，4。zipStorePrevEntryLength函数将修改p指向的entry的prevlen字段。另外，这里区分一下zipEntrySafe和zipEntry两个函数。两者都是将当前指向的entry元素解析为一个zlentry结构体，方便后面操作。区别在于前者是安全的，后者是不安全的。前者在解码的过程中会检查是否访问了ziplist所属内存空间外的内存，而后者假设传入的指向entry的指针p已经是检测过并且安全的，所以在解码的过程中不会额外检测。前者若检测到非法越界访问内存空间，则返回0，否则返回1。

set_tail字段用来修改对最后一个entry的偏移。最后就是移动被删除entry后面entry的内容到前面。

```c
if (p[0] != ZIP_END) {
    /*如果p指向的不是列表尾部，则计算当前p指向的元素的prevlen字段在删除前后的变化
     * */
    nextdiff = zipPrevLenByteDiff(p,first.prevrawlen);

    p -= nextdiff;
    assert(p >= first.p && p<zl+zlbytes-1);
    zipStorePrevEntryLength(p,first.prevrawlen);

    set_tail = intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))-totlen;

    assert(zipEntrySafe(zl, zlbytes, p, &tail, 1));
    if (p[tail.headersize+tail.len] != ZIP_END) {
        set_tail = set_tail + nextdiff;
    }

    size_t bytes_to_move = zlbytes-(p-zl)-1;
    memmove(first.p,p,bytes_to_move);
} else {
    set_tail = (first.p-zl)-first.prevrawlen;
}
```

**第三步**，重新分配压缩列表的空间。

```C
offset = first.p-zl;
zlbytes -= totlen - nextdiff;
zl = ziplistResize(zl, zlbytes);
p = zl+offset;
```

第四步，设置列表的长度，设置表尾元素偏移，并进行级联更新。

```c
ZIPLIST_INCR_LENGTH(zl,-deleted);

assert(set_tail <= zlbytes - ZIPLIST_END_SIZE);
ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(set_tail);

if (nextdiff != 0)
    zl = __ziplistCascadeUpdate(zl,p);
```



最后，这里简单提一下压缩列表的级联更新。可以大致分为三个步骤。

- 遍历一次更新位置的列表后面部分，直到一个entry不需要更新prevlen字段。对每个字段计算需要的额外空间。
- 更新tail offset。
- 从后往前，更新每个需要级联更新的entry的prevlen字段。







