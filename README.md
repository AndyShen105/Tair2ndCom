# 前言
这也算是我第一次参加这个比赛，学习到了很多知识，最终排名时15，成绩是60s，工作了时间是真的紧张阿，怀念上学的时候。

# 赛题
第二届数据库性能大赛将聚焦以Redis为代表的内存数据库技术，国内目前该领域影响力第一的是阿里云自研品牌Tair。本次大赛将结合Tair的最佳实践，并借力傲腾数据中心级持久内存技术，挑战在持久内存AEP上Keyvalue的性能极限。简而言之，就是将Aep应用到tair上。具体赛题见：https://tianchi.aliyun.com/competition/entrance/531820/introduction?spm=5176.14154004.J_3902846700.2.31fe5699LXl3th

# 知识储备
- Aep的结构介绍：https://software.intel.com/content/www/us/en/develop/videos/overview-of-the-new-intel-optane-dc-memory.html
- PMDK的介绍：https://pmem.io/pmdk/
- 性能测试：https://developer.aliyun.com/article/770338?groupCode=aliyundb

此外，我们还可以看一下官方的编成指南《Programming Persistent Memory》，今年FAST2020上有有一篇experiment paper对aep的性能做了测试。

# 初赛
## 测试指标
评测程序会调用参赛选手的接口，启动16个线程进行写入和读取数据，最终统计读写完指定数目所用的时间，按照使用时间的从低到高排名。
评测分为2个阶段：
1）程序正确性验证，验证数据读写的正确性（复赛会增加持久化能力的验证），这部分的耗时不计入运行时间的统计
2）初赛性能评测
引擎使用的内存和持久化内存限制在 4G Dram和 74G Aep。
每个线程分别写入约48M个Key大小为16Bytes，Value大小为80Bytes的KV对象，接着以95：5的读写比例访问调用48M次。其中95%的读访问具有热点的特征，大部分的读访问集中在少量的Key上面。

## 思路
### 一些关键点
首先我们从测试指标中可以获取以下几点信息：
- 写入量为48  * 16 * 96 * 1024*1024=72G 如果存储其他链接信息，本身是很容易超过Aep的总容量74G
- 初赛不考虑持久化，因此我们不需要强制flush，只需要把aep当作一个内存来用
- 读是热点读，后期我统计了一下只读了7000多个key
- 读写阶段只是update，纯写阶段只有小于百分之0.2的重复key

### 数据结构
#### 单条记录
对于每个写入的记录，我们存储的结构为：
KEY + NEXT_POS + VALUE
整体长度为16+4+80=100之所以这样存主要是有以下几点考虑：
- key和val都是定长，我们不需要存储一些len相关的信息
- 主题的索引采取的是HASH的链式存储，因此每条记录需要保存下一个enrty的position，
通过pos我们可以获取到数据的真实位置即：

```bash
address = pos * 100
```
这里的100为数据的长度，采取这种存储方式宗的数据量为75G，也就是大概有1G的数据存储到内存中，这时内存剩下3G供我们进行其他的操作

### 索引结构
上文说过，这里采用的是数组+链表的形式，因此在内存中 我们仅仅保存了链表的头部，每个元素我们称为Entry。

```cpp
class Entry {
 public:
  Entry() : head(UINT32_MAX){};
  ~Entry() = default;

  KEY_INDEX_TYPE getHead() const {
    return this->head.load(std::memory_order_relaxed);
  }
  // set head and return the old one
  KEY_INDEX_TYPE setHead(const uint32_t sn) {
    return this->head.exchange(sn, std::memory_order_relaxed);
  }

 public:
  std::atomic<KEY_INDEX_TYPE> head;
};
```
每个enrty中只保存了链表头的pos，并通过atomic来保证更新的一致性。每个enrty占用4个字节，为了减少哈希冲突，我们将这个数组的大小设置成400000000，占用了1.5G

### 流程
这里我们主要有两点优化：
- 写入threadlocal化，为了减少线程竞争，这里将aep均匀的分为16个Block，因为写入的量是均分的
- 写入buffer化，即对于每个线程的写入每k个组成一个buffer 写入到aep
- 在纯写入阶段采用append的方式写入数据，这样尽管存在重复数据，但是因为数据结构的缘故并不会读取到旧的数据

### 未考虑的点
- 内存对齐，在复赛时发现内存对齐对写入效率影响很大
