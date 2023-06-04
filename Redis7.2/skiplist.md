# Redis7.2 源码阅读笔记 —— SkipList

跳表是Redis中的有序集合ZSet的底层实现之一。

跳表这种数据结构可以和数组、链表、AVL树和红黑树等做比较学习。其中数组查询速度很快，但插入和删除操作时间复杂度比较高。链表很适合增删数据，但是不适合查找。AVL树和红黑树都是自动平衡二叉树，增删改查的效率都不错，区别在于保持平衡的策略，AVL树策略比较复杂，但是能保证更加平衡的结构，而红黑树对平衡性舍弃了一些，但提高了效率，二者的共同缺点在于实现的复杂。

除了上面提到的，当然还有跳表，总体来说它的效率堪比两种平衡二叉树，同时实现更加简单。

这里对跳表不做详细介绍，只是简单提一下，方便理解Redis中的跳表的实现。

## 跳表

首先回忆链表的实现，插入和删除只需要找到目标位置，然后改变前节点的指针和插入节点的指针即可。但是查找的时间时最坏情况需要遍历整个链表。为了减少遍历的节点，可以给链表加上“索引”。这里的索引是指在链表的上层加上额外的链表，但是节点更加稀疏，并且上下层的节点值是相同的，这样在上层检索到“下一个节点大于查找值的当前节点后”到下一层继续查找，通过这样的过程来减少遍历到的节点数。

这里只是对跳表的原理进行了简述，详情可以去看网上的其他资料。

## Redis中的SkipList

在Redis中，跳表的结构体分为两部分，zskiplist和zskiplistnode，前者即跳表结构体，后者是方便实现而定义的跳表节点结构体，两部分都在server.h文件中定义，代码如下：

```c
/* ZSETs use a specialized version of Skiplists */
typedef struct zskiplistNode {
    sds ele;
    double score;
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned long span;
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;
```

首先看zskiplist结构体，一共有四个成员，header和tail分别是指向跳表头节点和尾节点的指针，length是当前跳表包含的元素个数（最底层，不包含头节点），level是当前跳表的最高层数（不包含头节点）。

然后是链表节点结构体，一共有四个部分，首先是ele成员，是sds类型的，是真正存储的数据。score是排序的分数，当多个节点的score分数相同，则会按照ele的字典序排序。backword是指向前一个节点的指针（注意，头节点和第一个节点的backword指针为NULL）。最后一个元素level是一个柔性数组，数组中每个元素都是一个zskiplistLevel结构体，每个结构体包含两个元素——forward（指向当前层的下一个节点）和span（表示当前节点和forward指向的节点之间跳过的节点个数）。

## SkipList的创建

创建函数的定义在server.h中，即zslCreate，但是在t_zset.c文件中实现。代码如下。

```c
zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;

    zsl = zmalloc(sizeof(*zsl));
    zsl->level = 1;
    zsl->length = 0;
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;
        zsl->header->level[j].span = 0;
    }
    zsl->header->backward = NULL;
    zsl->tail = NULL;
    return zsl;
}
```

对上面的创建代码进行分析。首先分配一个zskiplist结构体的内存大小，并将高度和长度分别设置为1和0。然后创建一个头节点，高度为ZSKIPLIST_MAXLEVEL，定义在server.h中，为32，所以头节点的高度为32，score为0，ele为NULL，同样的backword默认为NULL。

## SkipList的插入操作

插入操作比较复杂， 这里分开解读。

首先是确定插入的位置。使用x作为临时节点，当x的下一个节点的score小于插入值的score，或者score相等但是下一个节点的字典序小于插入节点，则将x置为下一个节点，并且更新rank。每一层的最后都会设置update数组。

update数组是用于记录插入节点每一层的前一个节点的，rank用于记录当前层从header节点到update[i]节点的步长。

```c
zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
unsigned long rank[ZSKIPLIST_MAXLEVEL];
int i, level;

serverAssert(!isnan(score));
x = zsl->header;
for (i = zsl->level-1; i >= 0; i--) {
    /* store rank that is crossed to reach the insert position */
    rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
    while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                sdscmp(x->level[i].forward->ele,ele) < 0)))
    {
        rank[i] += x->level[i].span;
        x = x->level[i].forward;
    }
    update[i] = x;
}
```

然后是调整跳表的高度，代码如下。首先会调用zslRandomLevel函数计算插入节点的随机高度。该函数的代码也放在下面。具体逻辑是循环生成随机数，然后和threhold比较，如果小于阈值，则加1并继续这个过程，最后返回的高度要小于等于32。

计算出当前节点的高度之后，如果比插入之前的跳表的高度大，则更新rank和update数组。

```c
level = zslRandomLevel();
if (level > zsl->level) {
    for (i = zsl->level; i < level; i++) {
        rank[i] = 0;
        update[i] = zsl->header;
        update[i]->level[i].span = zsl->length;
    }
    zsl->level = level;
}

int zslRandomLevel(void) {
    static const int threshold = ZSKIPLIST_P*RAND_MAX;
    int level = 1;
    while (random() < threshold)
        level += 1;
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```

第三步是创建节点并插入，代码如下。首相创建一个zskiplistNode结构体，并对其进行插入，更新每层的前一个节点的forward指针和span。

```c
x = zslCreateNode(level,score,ele);
for (i = 0; i < level; i++) {
    x->level[i].forward = update[i]->level[i].forward;
    update[i]->level[i].forward = x;

    /* update span covered by update[i] as x is inserted here */
    x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
    update[i]->level[i].span = (rank[0] - rank[i]) + 1;
}
```

最后一段代码是对backword指针、tail指针和跳表长度进行调整，代码就不展示了。

## SkipList的删除操作

该功能有zslDelete函数实现。声明在server.h中，实现在t_zset.c文件中。首先，根据官方的注释，该函数在成功找到元素并删除后返回1，否则返回0。并且，调用者可以选择是否释放掉删除节点的内存，还是将该节点返回给调用者进行复用。

代码如下。首先第一段很明显是在查找要删除的目标节点，并利用update数组记录删除节点的每一层的前一节点。

找到节点后，会同时比较ele和score元素，都相等才会进行删除。如果node参数为NULL则释放对应内存，否则通过node返回被删除节点。

```c
int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                     sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    /* We may have multiple elements with the same score, what we need
     * is to find the element with both the right score and object. */
    x = x->level[0].forward;
    if (x && score == x->score && sdscmp(x->ele,ele) == 0) {
        zslDeleteNode(zsl, x, update);
        if (!node)
            zslFreeNode(x);
        else
            *node = x;
        return 1;
    }
    return 0; /* not found */
}
```

上面函数中用到的zslDeleteNode函数代码如下，同样可以分段理解，首先是更新每一层前一节点的指针和span元素。然后调整跳表的tail指针以及长度、高度。

```C
void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) {
    int i;
    for (i = 0; i < zsl->level; i++) {
        if (update[i]->level[i].forward == x) {
            update[i]->level[i].span += x->level[i].span - 1;
            update[i]->level[i].forward = x->level[i].forward;
        } else {
            update[i]->level[i].span -= 1;
        }
    }
    if (x->level[0].forward) {
        x->level[0].forward->backward = x->backward;
    } else {
        zsl->tail = x->backward;
    }
    while(zsl->level > 1 && zsl->header->level[zsl->level-1].forward == NULL)
        zsl->level--;
    zsl->length--;
}
```

zskiplist的分析暂时就到这里为止，当然Redis还为该数据结构实现了许多其他的功能方便上层调用，但是基本的思想都已经通过创建、插入和删除操作分析了，理解了这三个基本操作应该就能更好地对其他部分源码阅读和理解。



























