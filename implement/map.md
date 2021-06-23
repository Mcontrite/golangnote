# Golang map 的底层实现

## hash函数

Go语言中map采用的是哈希查找表，由一个key通过哈希函数得到哈希值(64位系统中就生成一个64bit的哈希值)，根据哈希值确定key-value落在哪个bucket的哪个cell。golang使用的hash算法和CPU有关，如果cpu支持aes，那么使用aes hash，否则使用memhash。当有多个哈希映射到相同的的桶中时，使用链表解决哈希冲突。

## hmap

可以理解为 header of map的缩写，即map数据结构的入口。

```go
type hmap struct {
    count      int            //元素个数，调用len(map)时直接返回
    flags      uint8          //标志map当前状态,正在删除元素、添加元素.....
    B          uint8          //单元(buckets)的对数 B=5表示能容纳32个元素，再多就要hashGrow了
    noverflow  uint16         //单元(buckets)溢出数量，如果一个单元能存8个key，此时存储了9个，溢出了，就需要再增加一个单元
    hash0      uint32         //随机哈希种子,可以防止哈希碰撞攻击
    buckets    unsafe.Pointer //指向单元(buckets)数组,大小为2^B，可以为nil
    oldbuckets unsafe.Pointer //扩容的时候，buckets长度会是oldbuckets的两倍
    nevacute   uintptr        //指示扩容进度，小于此buckets迁移完成
    extra      *mapextra      //可以减少GC扫描，当key和value都可以inline的时候就会用这个字段
}

```

## mapextra

存储key和value都不是指针类型的map，并且大小都小于128字节，这样可以避免GC扫描整个map。

```go
// 如果 key 和 value 都不包含指针，并且可以被 inline(<=128 字节)
// 使用 extra 来存储 overflow bucket，这样可以避免 GC 扫描整个 map
// 然而 bmap.overflow 也是个指针。这时候我们只能把这些 overflow 的指针
// 都放在 hmap.extra.overflow 和 hmap.extra.oldoverflow 中了
// overflow 包含的是 hmap.buckets 的 overflow 的 bucket
// oldoverflow 包含扩容时的 hmap.oldbuckets 的 overflow 的 bucket
```
```go
type mapextra struct {
    overflow       *[]*bmap
    oldoverflow    *[]*bmap

    // 指向空闲的 overflow bucket 的指针
    nextOverflow *bmap
}

```

## bmap

可以理解为buckets of map的缩写，它就是map中bucket的本体，即存key和value数据的“桶”。

每个bmap最多存8个key，每个key落在桶的位置有hash出来的结果的高8位决定。只存放8个key-value可以减少存储对象的数量，减轻内存管理的负担，有利于GC。

tophash 存储的是哈希函数算出的哈希值的高八位，用来加快索引。把高八位存储起来，不用完整比较key就能过滤掉不符合的key，加快查询速度。如果一个哈希值的高8位和存储的高8位相符合，再去比较完整的key值，进而取出value。

```go
type bmap struct {
    tophash [bucketCnt]uint8
}

//实际上编辑期间会动态生成一个新的结构体
type bmap struct {
    topbits  [8]uint8		// tophash 是 hash 值的高 8 位
    keys     [8]keytype		// 每个桶最多可以装8个key
    values   [8]valuetype	// 8个key分别有8个value一一对应
    pad      uintptr
    overflow uintptr		// 发生哈希碰撞之后创建的overflow bucket
}
```

bmp的内部组成如下：

![img](https://img2020.cnblogs.com/blog/1206020/202004/1206020-20200424152610842-674855726.png)

HOB Hash就是tophash，每个桶可以存储8对key-value，存储结构不是key/value/key/value...，而是key/key..value/value，这样可以减少pad字段节省内存空间。比如 map[int64]int8 如果以key-value的形式存储就必须在每个value后面添加padding7个字节，以上图的形式只需要在最后一个value后面添加padding就可以了。

## 内存布局

可以看出`hmap`和`bucket`的关系是这样的：

<img src="https://img2018.cnblogs.com/blog/1199549/201906/1199549-20190622232825554-644793812.png" alt="img" style="zoom: 67%;" />

而bucket又是一个链表，所以，整体的结构应该是这样的：

<img src="https://img2018.cnblogs.com/blog/1199549/201906/1199549-20190622233114658-1984741865.png" alt="img" style="zoom: 67%;" />

整体的内存布局

![img](https://upload-images.jianshu.io/upload_images/2472495-7be0b995dd41b9a6.png?imageMogr2/auto-orient/strip|imageView2/2/w/946/format/webp)

## 扩容

**装填因子是否大于阈值**

1. 装填因子 = 元素个数/桶个数=map长度 / 2^B(这是代表bmap数组的长度，B是取的低位的位数)。
2. 加载因子越小，说明空间空置率高使用率小，加载因子越大，说明空间利用率高但是产生冲突的几率也高了。
3. 阈值是6.5，6.5是经过测试后选取的一个平衡值。大于6.5时说明桶快要装满，需要扩容。

**overflow bucket是否太多**

1. bucket的数量 < 2^15，但overflow bucket的数量大于桶数量。
2. bucket的数量 >= 2^15，但overflow bucket的数量大于2^15。

**扩容方法**

1. 双倍扩容：装载因子多大，直接翻倍，B+1；但不是申请一块内存立马开始拷贝，而是每一次访问旧的buckets就迁移一部分，直到完成，旧bucket被GC回收。
2. 等量扩容：重新排列，极端情况下重新排列也解决不了，map成了链表，性能大大降低；哈希种子hash0的设置，可以降低此类极端场景的发生。

如下图所示：扩容时map结构体中，会保存旧的数据，和新生成的数组

<img src="https://img2018.cnblogs.com/blog/1199549/201906/1199549-20190622231255790-1061374322.png" alt="img" style="zoom: 50%;" />

上面部分代表旧的有数据的bucket，下面部分代表新生成的新的bucket。蓝色代表存有数据的bucket，橘黄色代表空的bucket。扩容时map并不会立即把新数据做迁移，而是当访问原来旧bucket的数据的时候，才把旧数据做迁移，如下图：

 <img src="https://img2018.cnblogs.com/blog/1199549/201906/1199549-20190622231311335-649924449.png" alt="img" style="zoom: 50%;" />

 注意：这里并不会直接删除旧的bucket，而是把原来的引用去掉，利用GC清除内存。

## 查找

哈希函数会将传入的key值进行哈希运算，得到一个唯一的值。go语言把生成的哈希值一分为二，比如一个key经过哈希函数，生成的哈希值为：`8423452987653321`，go语言会这它拆分为`84234529`，和`87653321`。那么，前半部分就叫做**高位哈希值**，后半部分就叫做**低位哈希值**。其中**低位哈希用来判断桶位置，高位哈希用来确定在桶中哪个cell**。

1. 低位哈希就是哈希值的低B位，hmap结构体中的B，比如B为5，2^5=32，即该map有32个桶，只需要取哈希值的低5位就可以确定当前key-value落在哪个桶(bucket)中；
2. 高位哈希即tophash，是指哈希值的高8bits，根据tophash来确定key在桶中的位置；
3. 当前bucket未找到则查找对应的overflow bucket；
4. 对应位置有数据则对比完整的哈希值，确定是否是要查找的数据；
5. 如果当前处于map进行了扩容，处于数据搬移状态，则优先从oldbuckets查找。


hash结果的低位用于选择把KV放在bmap数组中的哪一个bmap中，高位用于key的快速预览，用于快速试错。当不同的key根据哈希得到的tophash和低位hash都一样，发生哈希碰撞，这个时候就就需要把key-value对存储在overflow bucket（溢出桶），overflow pointer就是指向overflow bucket的指针。

如果overflow bucket也溢出了呢？那就再给overflow bucket新建一个overflow bucket，用指针串起来就形成了链式结构，map本身有2^B个bucket，只有当发生哈希碰撞后才会在bucket后链式增加overflow bucket。

![这里写图片描述](https://static.studygolang.com/180826/7d806b2a30f30d85e3ee65fb25929263.png)

如果在bucket中没有找到，此时如果overflow不为空，那么就沿着overflow继续查找，如果还是没有找到，那就从别的key槽位查找，直到遍历所有bucket。遍历所有bucket如下：

![img](https://img2020.cnblogs.com/blog/1206020/202004/1206020-20200427143851450-1714992347.png)


## 插入

1. 根据key计算出哈希值
2. 根据哈希值低位确定所在bucket
3. 根据哈希值高8位确定在bucket中的cell存储位置
4. 查找该key是否存在，已存在则更新，不存在则插入

## 删除

1. 如果key是一个指针类型的，则直接将其置为空，等待GC清除；
2. 如果是值类型的，则清除相关内存。
3. 同理，对value做相同的操作。
4. 最后把key对应的高位值对应的数组index置为空。

## map无序

map的本质是散列表，而map的增长扩容会导致重新进行散列，这就可能使map的遍历结果在扩容前后变得不可靠，Go设计者为了让大家不依赖遍历的顺序，故意在实现map遍历时加入了随机数，让每次遍历的起点即起始bucket的位置不一样，即不让遍历都从bucket0开始，所以即使未扩容时我们遍历出来的map也总是无序的。



# map 底层流程图

### 1. 数据结构及内存管理

hashmap的定义位于 src/runtime/hashmap.go 中，首先我们看下hashmap和bucket的定义：

```go
type hmap struct {
    count     int    // 元素的个数
    flags     uint8  // 状态标志
    B         uint8  // 可以最多容纳 6.5 * 2 ^ B 个元素，6.5为装载因子
    noverflow uint16 // 溢出的个数
    hash0     uint32 // 哈希种子

    buckets    unsafe.Pointer // 桶的地址
    oldbuckets unsafe.Pointer // 旧桶的地址，用于扩容
    nevacuate  uintptr        // 搬迁进度，小于nevacuate的已经搬迁
    overflow *[2]*[]*bmap 
}
```

其中，overflow是一个指针，指向一个元素个数为2的数组，数组的类型是一个指针，指向一个slice，slice的元素是桶(bmap)的地址，这些桶都是溢出桶；为什么有两个？因为Go map在hash冲突过多时，会发生扩容操作，为了不全量搬迁数据，使用了增量搬迁，[0]表示当前使用的溢出桶集合，[1]是在发生扩容时，保存了旧的溢出桶集合；overflow存在的意义在于防止溢出桶被gc。

```rust
// A bucket for a Go map.
type bmap struct {
    // 每个元素hash值的高8位，如果tophash[0] < minTopHash，表示这个桶的搬迁状态
    tophash [bucketCnt]uint8
    // 接下来是8个key、8个value，但是我们不能直接看到；为了优化对齐，go采用了key放在一起，value放在一起的存储方式，
    // 再接下来是hash冲突发生时，下一个溢出桶的地址
}
```

tophash的存在是为了快速试错，毕竟只有8位，比较起来会快一点。

从定义可以看出，不同于STL中map以红黑树实现的方式，Golang采用了HashTable的实现，解决冲突采用的是链地址法。也就是说，使用数组+链表来实现map。特别的，对于一个key，几个比较重要的计算公式为:

| key  |                  hash                   |                  hashtop                  |                     bucket index                      |
| :--: | :-------------------------------------: | :---------------------------------------: | :---------------------------------------------------: |
| key  | hash := alg.hash(key, uintptr(h.hash0)) | top := uint8(hash >> (sys.PtrSize*8 - 8)) | bucket := hash & (uintptr(1)<<h.B - 1)，即 hash % 2^B |

例如，对于B = 3，当hash(key) = 4时， hashtop = 0， bucket = 4，当hash(key) = 20时，hashtop = 0， bucket = 4；这个例子我们在搬迁过程还会用到。

内存布局类似于这样：

![img](https:////upload-images.jianshu.io/upload_images/7515493-7b4fa8bd0db2a5f2.png?imageMogr2/auto-orient/strip|imageView2/2/w/971/format/webp)

hashmap-buckets

### 2. 创建 - makemap

map的创建比较简单，在参数校验之后，需要找到合适的B来申请桶的内存空间，接着便是穿件hmap这个结构，以及对它的初始化。

![img](https:////upload-images.jianshu.io/upload_images/7515493-773c599bcc4a0559.png?imageMogr2/auto-orient/strip|imageView2/2/w/634/format/webp)

makemap

### 3. 访问 - mapaccess

对于给定的一个key，可以通过下面的操作找到它是否存在

![img](https:////upload-images.jianshu.io/upload_images/7515493-599f9d40d5c56e61.png?imageMogr2/auto-orient/strip|imageView2/2/w/987/format/webp)

image.png

方法定义为

```go
// returns key, if not find, returns nil
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer 

// returns key and exist. if not find, returns nil, false
func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool)

// returns both key and value. if not find, returns nil, nil
func mapaccessK(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, unsafe.Pointer)
```

可见在找不到对应key的情况下，会返回nil

### 4. 分配 - mapassign

为一个key分配空间的逻辑，大致与查找类似；但增加了写保护和扩容的操作；注意，分配过程和删除过程都没有在oldbuckets中查找，这是因为首先要进行扩容判断和操作；如下：

![img](https:////upload-images.jianshu.io/upload_images/7515493-54c06b9844da39bd.png?imageMogr2/auto-orient/strip|imageView2/2/w/993/format/webp)

assign

```
扩容是整个hashmap的核心算法，我们放在第6部分重点研究。
```

新建一个溢出桶，并将其拼接在当前桶的尾部，实现了类似链表的操作：

```go
// 获取当前桶的溢出桶
func (b *bmap) overflow(t *maptype) *bmap {
    return *(**bmap)(add(unsafe.Pointer(b), uintptr(t.bucketsize)-sys.PtrSize))
}

// 设置当前桶的溢出桶
func (h *hmap) setoverflow(t *maptype, b, ovf *bmap) {
    h.incrnoverflow()
    if t.bucket.kind&kindNoPointers != 0 {
        h.createOverflow()
        //重点，这里讲溢出桶append到overflow[0]的后面
        *h.overflow[0] = append(*h.overflow[0], ovf)
    }
    *(**bmap)(add(unsafe.Pointer(b), uintptr(t.bucketsize)-sys.PtrSize)) = ovf
}
```

### 5. 删除 - mapdelete

删除某个key的操作与分配类似，由于hashmap的存储结构是数组+链表，所以真正删除key仅仅是将对应的slot设置为empty，并没有减少内存；如下：

![img](https:////upload-images.jianshu.io/upload_images/7515493-a3221dbfcd6249ab.png?imageMogr2/auto-orient/strip|imageView2/2/w/853/format/webp)

mapdelete

### 6. 扩容 - growWork

首先，判断是否需要扩容的逻辑是

```go
func (h *hmap) growing() bool {
    return h.oldbuckets != nil
}
```

何时h.oldbuckets不为nil呢？在分配assign逻辑中，当没有位置给key使用，而且满足测试条件(装载因子>6.5或有太多溢出通)时，会触发hashGrow逻辑：

```go
func hashGrow(t *maptype, h *hmap) {
    //判断是否需要sameSizeGrow，否则"真"扩
    bigger := uint8(1)
    if !overLoadFactor(int64(h.count), h.B) {
        bigger = 0
        h.flags |= sameSizeGrow
    }
        // 下面将buckets复制给oldbuckets
    oldbuckets := h.buckets
    newbuckets := newarray(t.bucket, 1<<(h.B+bigger))
    flags := h.flags &^ (iterator | oldIterator)
    if h.flags&iterator != 0 {
        flags |= oldIterator
    }
    // 更新hmap的变量
    h.B += bigger
    h.flags = flags
    h.oldbuckets = oldbuckets
    h.buckets = newbuckets
    h.nevacuate = 0
    h.noverflow = 0
        // 设置溢出桶
    if h.overflow != nil {
        if h.overflow[1] != nil {
            throw("overflow is not nil")
        }
// 交换溢出桶
        h.overflow[1] = h.overflow[0]
        h.overflow[0] = nil
    }
}
```

OK，下面正式进入重点，扩容阶段；在assign和delete操作中，都会触发扩容growWork：

```go
func growWork(t *maptype, h *hmap, bucket uintptr) {
    // 搬迁旧桶，这样assign和delete都直接在新桶集合中进行
    evacuate(t, h, bucket&h.oldbucketmask())
        //再搬迁一次搬迁过程中的桶
    if h.growing() {
        evacuate(t, h, h.nevacuate)
    }
}
```

#### 6.1 搬迁过程

一般来说，新桶数组大小是原来的2倍(在!sameSizeGrow()条件下)，新桶数组前半段可以"类比"为旧桶，对于一个key，搬迁后落入哪一个索引中呢？

```bash
假设旧桶数组大小为2^B， 新桶数组大小为2*2^B，对于某个hash值X
若 X & (2^B) == 0，说明 X < 2^B，那么它将落入与旧桶集合相同的索引xi中；
否则，它将落入xi + 2^B中。
```

例如，对于旧B = 3时，hash1 = 4，hash2 = 20，其搬迁结果类似这样。

![img](https:////upload-images.jianshu.io/upload_images/7515493-061e88baf079da2f.png?imageMogr2/auto-orient/strip|imageView2/2/w/746/format/webp)

example.png

源码中有些变量的命名比较简单，容易扰乱思路，我们注明一下便于理解。

|       变量        |                            释义                            |
| :---------------: | :--------------------------------------------------------: |
|      x *bmap      |       桶x表示与在旧桶时相同的位置，即位于新桶前半段        |
|      y *bmap      | 桶y表示与在旧桶时相同的位置+旧桶数组大小，即位于新桶后半段 |
|      xi int       |                       桶x的slot索引                        |
|      yi int       |                       桶y的slot索引                        |
| xk unsafe.Pointer |                    索引xi对应的key地址                     |
| yk unsafe.Pointer |                    索引yi对应的key地址                     |
| xv unsafe.Pointer |                   索引xi对应的value地址                    |
| yv unsafe.Pointer |                   索引yi对应的value地址                    |

搬迁过程如下：

![img](https:////upload-images.jianshu.io/upload_images/7515493-58469b655416742d.png?imageMogr2/auto-orient/strip|imageView2/2/w/1007/format/webp)

evacuate

### 总结

到目前为止，Golang的map实现细节已经分析完毕，但不包含迭代器相关操作。通过分析，我们了解了map是由数组+链表实现的HashTable，其大小和B息息相关，同时也了解了map的创建、查询、分配、删除以及扩容搬迁原理。总的来说，Golang通过hashtop快速试错加快了查找过程，利用空间换时间的思想解决了扩容的问题，利用将8个key(8个value)依次放置减少了padding空间等等。

作者：Love语鬼
链接：https://www.jianshu.com/p/aa0d4808cbb8
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

 


