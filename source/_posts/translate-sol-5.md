---
title: "[翻译]Sol - 从零开始的MQTT broker - 第五部分：主题树"
url: translate-sol-5
date: 2023-12-28 10:43:27
categories:
- MQTT
tags:
- MQTT
- 翻译
- 网络编程
- 物联网
- C
- 数据结构
---

> 原文 [Sol - An MQTT broker from scratch. Part 5 - Topic abstraction](https://codepr.github.io/posts/sol-mqtt-broker-p5/)

<!--more-->

# 主题

在[第四部分]([../sol-mqtt-broker-p4](https://codepr.github.io/posts/sol-mqtt-broker-p4/))中，我们已经实现了两个数据结构，**哈希表**和**列表**。

在MQTT协议中，有一种名为 **主题(topic)** 的路由方式，主题本质上是一个字符串，用来将消息匹配到符合规则的客户端中。主题使用分层模型，遵守以下规则：

- 主题是一个UTF-8编码的字符串，最大长度为65535个字节
- `/` 用来区分不同的层级，就像文件系统一样
- `*` 是多层匹配通配符，例如使用 **foo/bar/\*** 可以匹配下列主题：
  - foo/bar
  - foo/bar/baz
  - foo/bar/bat/yop
- `+` 是单层匹配通配符，例如使用 **foo/+/baz** 可以匹配下列主题：
  - foo/bar/baz
  - foo/zod/baz
  - foo/nop/baz

主题和消息队列有一些类似的特性，但是主题更简单，更加轻量级，同时功能也更弱。

# 特里树定义

现在，我们开始定义我们的 **特里树(trie)** ，这将是我们用来存储主题的数据结构。特里树是这样一种结构，他的每一个节点都带有一个字符，从根到节点的所有字符就组成一个key，数据则是关联在key对应的位置上，在最糟糕的情况下，插入和查找的复杂度为 O(m)，其中 m 是键的长度。特里树的主要优点是可以方便的进行前缀匹配。

```c src/trie.h
#include <stdio.h>
#include <stdbool.h>
#include "list.h"

// 向用户提供类型 Trie
typedef struct trie Trie;

// 树节点, 包括一个子节点列表 children
// 如果是终端节点, 会在data中存储数据
struct trie_node {
    char chr;
    List *children;
    void *data;
};

// 特里树类型, 包括根节点和数据数量
struct trie {
    struct trie_node *root;
    size_t size;
};

// 创建一个新的字符节点
struct trie_node *trie_create_node(char);

// 创建一个新的特里树
struct trie *trie_create(void);

// 特里树初始化
void trie_init(Trie *);

// 当前大小
size_t trie_size(const Trie *);

/*
 * 叶子代表有关联数据的节点
 *           .
 *          / \
 *         h   s: s -> value
 *        / \
 *       e   k: hk -> value
 *      /
 *     l: hel -> value
 *
 * 上例中有三个键值对：
 * - s   -> value
 * - hk  -> value
 * - hel -> value
 */
void *trie_insert(Trie *, const char *, const void *);

bool trie_delete(Trie *, const char *);

// 查找节点, 查找成功时返回 true, 否则 false
// 第三个参数作为返回值, 提供指向查找结果的指针, 未找到时值为NULL
bool trie_find(const Trie *, const char *, void **);

// 释放一个节点, 同时更新size
void trie_node_free(struct trie_node *, size_t *);

void trie_release(Trie *);

// 通过一个前缀删除所有能匹配的节点
void trie_prefix_delete(Trie *, const char *);

// 使用 mapfunc 处理树中的所有节点, 第四个参数是 mapfunc 可使用的参数
void trie_prefix_map_tuple(Trie *, const char *, void (*mapfunc)(struct trie_node *, void *), void *);
```

# 特里树性能

关于树节点的实现，其实有很多种不同的方法，最简单的一种就是在每一个节点上使用固定长度的数组，数组的大小就是完整的字母表大小，例如这样：

```c
#define ALPHABET_SIZE 94

// 使用固定数组大小的树节点
struct trie_node {
    struct trie_node *children[ALPHABET_SIZE];
    void *data;
};
```

除了可以对key进行范围查询（用以实现通配符功能）这个最大的优点外，特里树的另一个巨大优点是他基于哈希表或者说 B-Tree 的性能优势，能够在进行插入、删除和搜索时保持最坏为 O(L) 的时间复杂度（L是查询键的长度）。但这是有代价的，最明显的缺陷就是结构体自身的内存消耗。

在上面的例子中，我们的字母表长度是96，意味着从`空格` 开始一直到 `~` 结束的96个代表不同字符的 `NULL` 指针都会被存在 `children` 中。在一个64位的机器上，每个指针需要使用8个byte，也就是说一个节点至少需要 96 * 8 = 768 个字节的空间。我们举个简单的例子：

- 插入一个key `foo`
- 插入一个key `foot`

此时我们的根节点 `f` 有一个非空指针 `o`，`o` 也有一个非空指针 `o`，这里储存着键 `foo` 对应的值。第二个 `o` 还有一个非空指针 `t`， 他将存储键 `foot` 对应的值。所以我们总共会有4个节点，这意味着我们会有 4 * 96 = 384 个指针，然后只使用了其中的4个，显然造成了很大的空间浪费。

当然，业界早有解决这个问题的方法，即减少空间浪费又保持着良好的时间复杂度性能，比如压缩特里树（compressed trie）和自适应特里树（adaptive trie）。

我们不去深入挖掘这些概念，就我们目前情况来看，可以想到三个解决方案：

- 在特里树结构体本身（非节点）添加一个动态列表，每个节点都必须拥有一个指向该列表的指针，和一个`char children_idx[ALPHABET_SIZE]`数组，数组中保存了自己的子节点在列表中的索引（这句话译者没有理解，原文：Use a single dynamic array (vector) in the Trie structure, each node must have a pointer to that vector and an array char children_idx[ALPHABET_SIZE] which store the index in the main vector for each children，如果你能理解，请告诉我，感谢）

- 使用基于子节点数量增长存储空间的节点，例如当子节点数量 <= 4 时，可以使用固定长度为4的数组，当子节点数量增长时将数组更换为更大的数组并且重新关联子节点。

- 将每个节点上的定长数组更换为 **链表**， 在每次插入操作后保持排序，这样每次搜索的平均性能为 O(n/2)，等同于 O(n)。

恰好我们在上个部分中实现了基于链表的列表，接下来就让我们使用第三种方案来实现我们的特里树。

# 特里树实现

```c src/trie.c
#include <assert.h>
#include <stdlib.h>
#include <string.h>
#include "list.h"
#include "trie.h"

// 合并两个输入的链表为一个链表, 并从小到大排序
// 要求输入的两个链表都是已经被从小到大排序
static struct list_node *merge_tnode_list(struct list_node *list1,
                                          struct list_node *list2) {
    struct list_node dummy_head = { NULL, NULL }, *tail = &dummy_head;
    // 每次都取 l1 或 l2 中较小的那个, 循环操作
    while (list1 && list2) {

        // 使用char比较大小
        char chr1 = ((struct trie_node *) list1->data)->chr;
        char chr2 = ((struct trie_node *) list2->data)->chr;

        struct list_node **min = chr1 <= chr2 ? &list1 : &list2;
        struct list_node *next = (*min)->next;
        tail = tail->next = *min;
        *min = next;
    }
    tail->next = list1 ? list1 : list2;
    return dummy_head.next;
}

// 被递归调用, 将链表按照从小到大顺序排序
struct list_node *merge_sort_tnode(struct list_node *head) {
    struct list_node *list1 = head;
    if (!list1 || !list1->next)
        return list1;
    // 从中分开
    struct list_node *list2 = bisect_list(list1);
    return merge_tnode_list(merge_sort_tnode(list1), merge_sort_tnode(list2));
}

// 通过给定的 val 搜索链表中的 trie_node, 最糟糕情况下的时间复杂度是 O(n)
static struct list_node *linear_search(const List *list, int value) {
    if (!list || list->len == 0)
        return NULL;
    for (struct list_node *cur = list->head; cur != NULL; cur = cur->next) {
        if (((struct trie_node *) cur->data)->chr == value)
            return cur;
        // 链表内部的节点是按照 chr 排序的，因此当大于 value 时无需继续搜索
        else if (((struct trie_node *) cur->data)->chr > value)
            break;
    }
    return NULL;
}

// 辅助比较函数, 传入两个 list_node, 当其中的 trie_node->chr 相等时返回0, 否则返回1
static int with_char(void *arg1, void *arg2) {
    struct trie_node *tn1 = ((struct list_node *) arg1)->data;
    struct trie_node *tn2 = ((struct list_node *) arg2)->data;
    if (tn1->chr == tn2->chr)
        return 0;
    return -1;
}

// 判断 trie_node 是否有子节点, 没有则认为 free
static bool trie_is_free_node(const struct trie_node *node) {
    return node->children->len == 0 ? true : false;
}

// 从 node 开始, 通过传入的 prefix, 找到对应的节点
static struct trie_node *trie_node_find(const struct trie_node *node, const char *prefix) {
    // 结果, 最初指向 node
    struct trie_node *retnode = (struct trie_node *) node;

    // 遍历 prefix 向下查找
    for (; *prefix; prefix++) {
        // O(n)
        struct list_node *child = linear_search(retnode->children, *prefix);
        // 没找到
        if (!child)
            return NULL;
        retnode = child->data;
    }
    return retnode;
}

// 创建一个新节点
struct trie_node *trie_create_node(char c) {
    struct trie_node *new_node = malloc(sizeof(*new_node));
    if (new_node) {
        new_node->chr = c;
        new_node->data = NULL;
        new_node->children = list_create(NULL);
    }
    return new_node;
}

// 创建并初始化特里树, 当前 size 0
Trie *trie_create(void) {
    Trie *trie = malloc(sizeof(*trie));
    trie_init(trie);
    return trie;
}

void trie_init(Trie *trie) {
    trie->root = trie_create_node(' ');
    trie->size = 0;
}

size_t trie_size(const Trie *trie) {
    return trie->size;
}

// 插入数据, 插入沿途的所有所需节点, 如果目标节点已经有数据则替换
// return 被插入的数据
// root 一般是根节点
// key 插入位置目标键
// data 要插入的数据内容
// size 带入 size, 如新增数据则+1
static void *trie_node_insert(struct trie_node *root, const char *key, const void *data, size_t *size) {
    struct trie_node *cursor = root;   // 上级节点
    struct trie_node *cur_node = NULL; // 当前节点
    struct list_node *tmp = NULL;      // 包裹当前节点的 list_node

    // 逐字符遍历 key
    for (; *key; key++) {
        // 我们使用一个 O(n) 复杂度的线性搜索器来搜索匹配的节点
        tmp = linear_search(cursor->children, *key);

        // 如果没有匹配, 我们会添加一个节点, 然后对所有的节点进行排序
        if (!tmp) {
            cur_node = trie_create_node(*key);
            cursor->children = list_push(cursor->children, cur_node);
            cursor->children->head = merge_sort_tnode(cursor->children->head);
        } else {
            // 匹配成功, 进入下一层匹配
            // 如果此时 key 已经被阅读完, 后续直接使用此节点
            cur_node = tmp->data;
        }
        cursor = cur_node;
    }

    // 如果新节点或此节点没有数据, 则记录size
    if (!cursor->data)
        (*size)++;
    cursor->data = (void *) data;
    return cursor->data;
}

// 删除 key 节点对应的数据, 同时递归的向上删除每一层不需要的节点（指既没有数据也没有子节点）
// return 节点本身是否可被删除(如果没有子节点就可以删除)
// node 寻找的起始节点, 一般用root
// key 匹配键
// size 当前树的大小
// found 返回是否删除成功
static bool trie_node_recursive_delete(struct trie_node *node, const char *key,
                                       size_t *size, bool *found) {
    if (!node)
        return false;

    // 字符串已经递归到达尾部的情况
    if (*key == '\0') {
        if (node->data) {
            // 标记成功找到数据
            *found = true;
            // 释放资源（以下为作者原代码，译者觉得这里重复操作了）
            if (node->data) {
                free(node->data);
                node->data = NULL;
            }
            free(node->data);
            node->data = NULL;
            // 记录size
            if (*size > 0)
                (*size)--;
            // 如果没有子节点, 标记需要被删除
            return trie_is_free_node(node);
        }
    } else {
        // 通过 key 逐字符匹配节点
        struct list_node *cur = linear_search(node->children, *key);
        if (!cur)
            return false;
        struct trie_node *child = cur->data;
        if (trie_node_recursive_delete(child, key + 1, size, found)) {

            // 从list中删除和当前剩余key后缀相同的节点
            struct trie_node t = {*key, NULL, NULL};
            struct list_node tmp = {&t, NULL};
            list_remove(node->children, &tmp, with_char);

            // 把被删除的节点释放掉
            trie_node_free(child, size);

            // 递归, 逐级向上删除可以被删除的节点
            return (!node->data && trie_is_free_node(node));
        }
    }
    return false;
}

// 从根节点开始寻找目标节点, 提供目标节点的数据
// return 是否查询成功
// root 一般传入根节点
// key 查询的键
// ret 返回值 (使用双指针因此当无数据时 *ret 可以为NULL)
static bool trie_node_search(const struct trie_node *root, const char *key, void **ret) {
    struct trie_node *cursor = trie_node_find(root, key);
    *ret = (cursor && cursor->data) ? cursor->data : NULL;
    return !*ret ? false : true;
}

// 插入数据
void *trie_insert(Trie *trie, const char *key, const void *data) {
    assert(trie && key);
    return trie_node_insert(trie->root, key, data, &trie->size);
}

// 删除某个节点的数据 (会递归的删除上层不需要的节点)
bool trie_delete(Trie *trie, const char *key) {
    assert(trie && key);
    bool found = false;
    if (strlen(key) > 0)
        trie_node_recursive_delete(trie->root, key, &(trie->size), &found);
    return found;
}

// 获得数据
bool trie_find(const Trie *trie, const char *key, void **ret) {
    assert(trie && key);
    return trie_node_search(trie->root, key, ret);
}

// 使用前缀删除所有匹配的内容
// 例如 prefix = "hello"
// 会删除这些内容： hello hellot helloworld
void trie_prefix_delete(Trie *trie, const char *prefix) {
    assert(trie && prefix);

    // 找到前缀对应的节点
    struct trie_node *cursor = trie_node_find(trie->root, prefix);
    if (!cursor)
        return;

    // 如果没有子节点, 就直接删除这个节点即可
    if (cursor->children->len == 0) {
        trie_delete(trie, prefix);
        return;
    }
    // 如果有子节点
    struct list_node *cur = cursor->children->head;
    // 遍历
    for (; cur; cur = cur->next) {
        // 递归的释放所有内容
        trie_node_free(cur->data, &(trie->size));
        // 并将子节点指针置空
        cur->data = NULL;
    }

    trie_delete(trie, prefix);
    list_clear(cursor->children, 1);
}

// 使用传入的函数处理每个节点, 将 node 作为首个节点逐层向下遍历, arg 允许作为函数的参数
static void trie_prefix_map_func2(struct trie_node *node,
                                  void (*mapfunc)(struct trie_node *, void *), void *arg) {
    if (trie_is_free_node(node)) {
        mapfunc(node, arg);
        return;
    }
    struct list_node *child = node->children->head;
    for (; child; child = child->next)
        trie_prefix_map_func2(child->data, mapfunc, arg);
    // node 本身也会被应用
    mapfunc(node, arg);
}

// 从前缀对应的节点开始调用 trie_prefix_map_func2
void trie_prefix_map_tuple(Trie *trie, const char *prefix,
                           void (*mapfunc)(struct trie_node *, void *), void *arg) {
    assert(trie);
    if (!prefix) {
        trie_prefix_map_func2(trie->root, mapfunc, arg);
    } else {
        // 找到key对应的节点
        struct trie_node *node = trie_node_find(trie->root, prefix);
        // 没有匹配到的节点
        if (!node)
            return;
        // 通过递归让node和所有的子节点都应用 mapfunc
        trie_prefix_map_func2(node, mapfunc, arg);
    }
}

// 递归的, 从node开始向下全部释放并删除
void trie_node_free(struct trie_node *node, size_t *size) {
    if (!node)
        return;

    // 这里的递归处理删除所有子节点
    if (node->children) {
        struct list_node *cur = node->children->head;
        for (; cur; cur = cur->next)
            trie_node_free(cur->data, size);
        list_release(node->children, 0);
        node->children = NULL;
    }

    // 译者并没有看明白这里？也许是某种编程技巧？
    if (node->data) {
        free(node->data);
        if (*size > 0)
            (*size)--;
    } else if (node->data) {
        free(node->data);
        if (*size > 0)
            (*size)--;
    }

    // 释放node本身
    free(node);
}

// 释放整个特里树
void trie_release(Trie *trie) {
    if (!trie)
        return;
    trie_node_free(trie->root, &(trie->size));
    free(trie);
}
```

# 结尾

写到这里，我们的工具基本上够用了。现在我们的项目又多了三个模块：

```text
sol/
 ├── src/
 │    ├── mqtt.h
 |    ├── mqtt.c
 │    ├── network.h
 │    ├── network.c
 │    ├── list.h
 │    ├── list.c
 │    ├── hashtable.h
 │    ├── hashtable.c
 │    ├── trie.h
 │    ├── trie.c
 │    ├── util.h
 │    ├── util.c
 │    ├── pack.h
 │    └── pack.c
 ├── CHANGELOG
 ├── CMakeLists.txt
 ├── COPYING
 └── README.md
```
