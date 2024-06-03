---
layout:       post
title:        "golang map 扩容过程中迭代器行为"
author:       "Quinlan"
header-style: text
catalog:      true
tags:
    - golang
---


在没有扩容发生时，for-range 逻辑比较简单，随机从 buckets 某个位置开始遍历即可，再次遍历到起始位置，表示遍历结束，用伪代码表示如下
```golang
startIdx := rand() % bucketsLen
wrapped := false    // 当遍历到 buckets 末尾, 从 idx=0 开始遍历时, wrapped=true
for idx := startIdx; ;idx++ {
    // 到末尾后从头开始, 再次到达起始位置, 表示环形遍历完成
    if idx == startIdx && wrapped {
        break
    }


}
```

# 为什么需要扩容？
map 常见的实现方式是哈希表，同时使用分离链接法解决键冲突的问题。golang 也采用了这种实现方式。整体的结构如下图示意

![map示意图](http://sehufdf0y.hn-bkt.clouddn.com/whiteboard_exported_image.png)

哈希表认为是 O(1) 复杂度的，但是当单个 slot 中存储的元素过多时，某些 key 的访问会退化为链表查找，即 O(n) 复杂度。所以这时通常需要对 map 进行扩容。即图中的 `buckets []*bmap` 数组的大小。数组的大小由`hmap.B`决定，大小为 2^B，每次扩容为之前的两倍，即 B=B+1。

> 这里表示不严谨，也存在 same size grow，这里暂且不表，可查看 github issue.

# 扩容状态
当扩容发生时，hmap 存在两个 `[]*bmap` 数组，分别为 buckets 和 oldBuckets，两者的长度不同，前者为后者的两倍。

![golang bucket evacated](http://sehufdf0y.hn-bkt.clouddn.com/golang-map-bucket-evacate.png)

golang 中 buckets 的长度一定的2的幂，所以根据 hash 值确定桶的位置时，采用的是幂运算
```golang
mask := shift(n)    // mask 是低n位全1的整数
hash := Hasher(key, seed)
idx := mask & hash  // 通过位与的操作, 可以快速完成取余的计算
```

以图中举例，扩容前 B=2，扩容后 B=3，hash 值的计算结果没有发生改变，只有位计算的多了一位，之前是根据低2位确定桶的位置，现在根据低3位确定桶的位置。<br>

因此根据第3比特位的数值不同，如果是0，则和之前的结果一致，如果是1，则向后偏移n位，n=2^old_B.

因此图中位置1的元素，被分流到了新 buckets 数组的 1 和 5 的位置。

# 扩容过程中的迭代器
整个迭代过程，可以先用伪代码简要描述
```golang
func iter() {
    for idx := range buckets {  // 遍历新 buckets 下标
        b := buckets[idx]
        oldB := getOldBucket()
        checking := false
        if !oldB.evacated() {
            checking = true
            b = oldB
        }

        // 分离链接法, 遍历所有的溢出桶
        for ; b != nil; b = b.overflow() {
            for elem := 0; elem < 8; elem++ {
                if checking {
                    // 检查旧 bucket 的元素是否会移动到当前遍历的 bucket，如果不是，不应该在这个 bucket 遍历时访问
                }
            }
        }
    }
}
```

在 `growing` 的过程中，最重要的时，如果当前 slot 的元素还没有迁移，则需要检查旧槽中的数据，检查旧 bucket 的元素是否会移动到当前遍历的 bucket，如果不是，不应该在这个 bucket 遍历时访问。

## 为什么要回溯旧 bucket 来访问元素?
growth 过程中，数据要么在新 buckets，要么在旧 buckets，数据只会存在唯一的一份，因此可以采用先遍历新 buckets，再遍历旧 buckets，或者返回来先旧后新，这样依然可以完成遍历的动作。

这种思路会存在一个问题，例如新 bucket 里只有一个槽，这个槽里这有一个元素，那么在每次遍历过程中，第一个元素就是固定的，这和 golang 对 iterator 是随机访问的设定不符。




