### 1、WET: Write Efficient Loop Tiling for Non-Volatile Main Memory

#### 1.1、本文提出了一种对矩阵乘法运算进行多层次的循环分片来有效地降低NVM主存的写次数

#### 1.2、本文列举了普通的矩阵乘法分片以及分治法的乘法分片，其目的都是为了让子矩阵的大小拟合L1缓存，以追求最大速度，却没有考虑到写次数。文章提出的WET方法使用双层次的分块，外层分块大小用于拟合LLC，内层分块大小拟合L1。

#### 1.3、在结果方面，写次数能有效地降低，并且不影响其原来的性能（甚至还稍微快了一点），写次数降低的比例为普通矩阵乘法分片的 insize/outsize 倍。

#### 1.4、以下列举出普通矩阵乘法的代码，分治法的乘法策略以及作者提出的WET矩阵乘法的代码。

* ```c++
  //普通矩阵乘法
  for (k2 = 0; k2 < n; k2 += tsize) 
      for (i2 = 0; i2 < n; i2 += tsize) 
          for (j2 = 0; j2 < n; j2 += tsize)
              for (i = i2; i < (i2 + tsize); i++) 
                  for (j = j2; j < (j2 + tsize); j++) {
                      sum = R[i][j];
                      for (k = k2; k < (k2 + tsize); k++) 
                          sum += A[i][k] * B[k][j];
                     	R[i][j] = sum;
                  }
  ```

* 分治法的策略![image-20211005163500996](essaySummary(10.4).assets/image-20211005163500996.png)

- ```c++
  //WET
  for (k3 = 0; k3 < n; k3 += outerTile) 
      for (i3 = 0; i3 < n; i3 += outerTile) 
          for (j3 = 0; j3 < n; j3 += outerTile) 
              for (k2 = 0; k2 < (k3 + outerTile); k2 += innerTile) 
                  for (i2 = 0; i2 < (i3 + outerTile); i2 += innerTile) 
                      for (j2 = 0; j2 < (j3 + outerTile); j2 += innerTile)
                          for (i = i2; i < (i3 + innerTile); i++) 
                              for (j = j2; j < (j3 + innerTile); j++) {
                                  sum = R[i][j];
                                  for (k = k2; k < (k3 + innerTile); k++) 
                                      sum += A[i][k] * B[k][j];
                                  R[i][j] = sum;
                              }
  ```

#### 1.5、对论文的思考：

* 论文的设计实际上并非很难想到，本文提出以前做矩阵计算只划分到L1存储大小，因为这样就可以极大地提升其速度，而再去划分其他的缓存提升并不多。在这种情况下，由于NVM内存对写次数有很大的限制，所以，我们在LLC上再增加一层子矩阵来聚集L1划分的更小矩阵的写操作，这样便可以减少写入NVM的次数。

* 其次，我发现本文提出的WET在设置循环方面具有能改进的地方，改进过后能将写入次数减少N/outsize 倍！

* 以下是我改进的算法代码：

  实际上也就是将k3的位置调整为i3和j3后面，这样可以大幅度减少NVM重复的写入

  ```c++
  //optimization
  for (i3 = 0; i3 < n; i3 += outerTile) 
      for (j3 = 0; j3 < n; j3 += outerTile) 
          for (k3 = 0; k3 < n; k3 += outerTile) 
              for (k2 = 0; k2 < (k3 + outerTile); k2 += innerTile) 
                  for (i2 = 0; i2 < (i3 + outerTile); i2 += innerTile) 
                      for (j2 = 0; j2 < (j3 + outerTile); j2 += innerTile)
                          for (i = i2; i < (i3 + innerTile); i++) 
                              for (j = j2; j < (j3 + innerTile); j++) {
                                  sum = R[i][j];
                                  for (k = k2; k < (k3 + innerTile); k++) 
                                      sum += A[i][k] * B[k][j];
                                  R[i][j] = sum;
                              }
  ```

### 2、A Wear-Leveling-Aware Fine-Grained Allocator for Non-Volatile Memory

#### 2.1、本文提出了一种对页面内部磨损的细粒度磨损均衡分配器。

#### 2.2、均衡器分为三个方面，分配、释放、重置

#### 2.3、分配提出了 Clockwise Best-Fit (CBF) 策略，具体实现是将page分成64个单元，最后一个单元存放元数据。元数据中记录page中空闲内存的信息，例如 单元空闲位映射、空闲单元总数、最大连续空闲单元，使用DRAM存储bucket桶按最大连续空闲内存用双链表连接。当需要分配内存时，按照best-fit算法从链表中按顺序拿出page分配。bucket桶在NVM内存中具有备份。

![image-20211008101615198](essaySummary(10.4).assets/image-20211008101615198.png)

![image-20211008101630496](essaySummary(10.4).assets/image-20211008101630496.png)

#### 算法实现：

![image-20211008101752393](essaySummary(10.4).assets/image-20211008101752393.png)

#### 2.4、释放操作，每次释放操作，就将元数据进行更改即可，并不改变 page在bucket中的位置。

#### 2.5、重置操作，由于释放操作的存在，使得桶内page内的空闲空间变多，需要重置每个桶的page。

![image-20211008103921534](essaySummary(10.4).assets/image-20211008103921534.png)

#### 文中当桶内页面的空闲平均值大于该桶的预设值K时重置该桶的page。

#### 2.6、思考与改进

* 首先是CBF分配策略，该种方法是作为Best-fit算法的改进，每次查找最接近需分配空间的最小连续空闲区域。该种方案为何比图中所说的其他分配方法好，我是不知道为什么的，第一个原因是我没有看过文中所说的其他分配方法，其次是这种分配方法很难直接证明，是否只能通过实验说明该种分配方法好？
* 释放操作只更改元数据，而不移动page在桶内的位置，这样能极大地减轻释放所带来的时间消耗，而不移动page的负面影响将在重置操作中得到解决。
* 重置操作文中说的是有一些问题的，首先文中的那个判定条件我认为在任何时候对于任何桶都是成立的，并没有什么用处。其次是文中并未说明检测该参数的时间等设置，如果检测时间间隔太短，会产生大量的时间消耗，如果太长，又会导致太多页面放置在不合适的桶内。
  * 首先得判定什么时候该检测：一、选择特定的时间检测，比如设置间隔100ms，然后对每个桶检测其是否需要重置操作。二、设定一个计数器，当释放操作数量达到一定阈值时，检测每个桶是否需要重置。
  * 其次是一个桶满足什么条件将对其进行重置操作。我认为是 当每个桶内的页面 平均每个页面的最大连续单元大于等于其预设值K + 1 时，对其进行操作，这样的话，该桶内的大部分页面都能转移到更大空闲内存的桶内，增加其效率。
