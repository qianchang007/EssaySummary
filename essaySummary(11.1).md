### 1、Write-Optimized Dynamic Hashing for Persistent Memory

#### 1.1、作者提出一种对PM友好的高效的动态哈希映射结构CCEH（Cacheline-Conscious Extendible Hashing）。

#### 1.2、CCEH采用三层结构：

- 第一层为最上层，叫做目录，目录用于映射段。
- 第二层是段，段内具有2^x^个桶，段内根据末位x比特来hash到每个桶中。
- 每个桶具有64字节，桶内可以存放key-value。
- 目录映射段是具有一定的关系的，简单地说，目录就是有2^a^个单位，段具有不大于2^a^个，每个目录都映射一个段，但是段可以被不止一个目录映射。假设该a值为2，目录有00、01、10、11，00目录->0段，01->1段，10,11目录->2段。当2段发生冲突时，我们可以增加一个段来解决冲突，并且不改变目录结构，即00目录->0段，01->1段，10目录->2段，11目录->3段。但是0段发生冲突时，我们就必须增加目录大小，使其扩大一倍，并且增加一个段。a被称为全局深度，还有一个局部深度定义为全局深度-log2(指向该段的目录数)，也就是说00目录->0段，01->1段，10,11目录->2段（等价于 最高位为1的目录映射为2段），两个位映射一个段，那该目录的局部深度就为2，如果是2段，只需要一个位1就能映射到2，那它的局部深度为1。
- 以下是图示：（全局深度为G，局部深度为L）

![image-20211104174618231](essaySummary.assets/image-20211104174618231.png)

#### 1.3、结构的动态扩大和缩小

- 当一个键-值结构按照hash到某个桶，但是桶中却已经满了时，就需要增加一个段结构了。段结构的拆分分为三个步骤：
  - 从被拆分段中hash键值对到新段中
  - 更新指向新段目录的指针与局部深度（从右到左）
  - 更新指向被拆分的段目录的局部深度（从右到左）
- 当需要将段进行合并时：
  - 从右段中hash键值对到左端中
  - 更新指向左段的局部深度（从左到右）
  - 更新指向右段的指针（从左到右）

#### 1.4、结构的并发

- 当需要更新桶元素时，使用读写锁
- 当需要更新段时，采用lazy deletion 或者采用 COW，采用lazy deletion能减少更新时间，但是会增加查询和插入时间，COW则相反。
- 更新目录结构采用锁结构更新

#### 1.5、装载因子

- 作者对该点提出了自己的思考，作者采用线性探测当在同一段上进行。作者对线性探测的距离做出限制，否则会极大地影响查询插入的性能。

### 2、ArchTM: Architecture-Aware, High Performance  Transaction for Persistent Memory

#### 2.1、作者提出两点PM微体系结构思路去优化事务方面的处理，提出ArchTM实现该设想。

- 避免256bytes（Optane PM write block）以下的写入
- 鼓励合并写，特别是顺序写；

#### 2.2、作者提出了以下五点措施来应用上述的两点优化思想。

- ##### 避免小写入

  1. Logless：使用CoW
  2. 最小化PM上的metadata修改，并保证crash consistency
  3. Scalablepersistent object referencing：ArchTM在DRAM上使用一个lookup table 快速定位PM数据对象的最新副本

- 鼓励合并顺序写

  4. 连续的分配请求分配连续的内存块
  5. 减少内存碎片。

#### 2.3、作者的实现思路

- 为了减少微小写入，ArchTM把metadata的内存管理和数据存储放在DRAM上，虽然会有很高的性能，但是还是会带来crash一致性的问题。这个问题的本质是metadata是事务状态和数据对象（文件）之间唯一的关联。ArchTM引入了一个注释机制来解决这个问题。这个注释机制的基本设计是：把数据对象元数据和TxID放到数据对象中，并且把TxID放到事务元数据中。这里的TxID是一个持久的，并不是动态分配的，而且可以起到连接数据对象和事务状态的作用。所以通过TxID，数据对象ID和数据大小，就可以很容易的定义数据对象，并且检测一致性。
- 为了鼓励顺序合并写，ArchTM尽可能保证连续的内存分配，因为根据观察，连续分配的数据对象很可能一起更新，但是目前的内存分配策略是管理一系列free list，每个list保存一定的连续内存空间，这样的分配可以避免大量的内存碎片，因此这是一个内存碎片和连续空间之间的tradeoff。所以ArchTM只使用一个free list，同时实现了一个尽可能避免碎片的机制。对于内存分配，使用单一的free list和一个recycle list来进行收集和合并空闲内存快。
