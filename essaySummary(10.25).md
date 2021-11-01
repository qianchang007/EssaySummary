### 1、Optimizing Systems for Byte-Addressable NVM by Reducing Bit Flipping

#### 1.1、本文作者对于BNVM的位翻转提出了一些优化。

#### 1.2、主要是以下三个方面：

- 对一些数据结构的位翻转进行优化（eg，double linked list， hash table， red-black tree）。
- 对编译器中的寄存器入栈部分进行了位翻转的优化。
- 指出内存分配过程也会有一些位翻转差异。

#### 1.3、简要说明这些优化的原理：

- 对于数据结构的优化都是围绕一个点进行的，就是利用异或操作将指针数目减少，并且现有指针也不增加其位翻转

  - 对于double linked list， 只采用一个指针代替原来的双指针，具体设计为将原来的双指针异或得到一个指针，头指针记录的是 `head ^ next`, 尾指针记录的是` tail ^ pre`。

  - 对于hash table，相当于是故技重施，使用linked list来避免冲突，对next指针设置为 `self ^ next`。

  - 对于red-black tree， 也是同理，一般一个节点存储三个指针，parent、left、right，可以缩减为两个，即为 `left ^ parent`, `right ^ parent`。

  - 下面代码是关于XOR优化双向列表和普通列表的插入和删除，自己写的，不一定对。

    ```c++
    //XOR linked list
    
    //insert
    tmp = malloc(node*);
    tmp->pointer = head ^ tmp;
    if (head != null)
    	head->pointer ^= head ^ tmp;
    head = tmp;
    //delete
    preNode = tail->pointer ^ tail;
    if (preNode != null)
        preNode->pointer ^= tail ^ preNode;
    if (head == tail)
        head = null;
    free(tail);
    tail = preNode;
    ```

    ```
    //conventional linked list
    
    //insert
    tmp = molloc(node*);
    tmp->pre = null;
    tmp->next = head;
    if (head != null) 
        head->pre = tmp;
    head = tmp;
    //delete
    preNode = tail->pre;
    if (preNode != null)
        preNode->next = null;
    if (head == tail)
        head = null;
    free(tail);
    tail = preNode;
    ```

    

- 其次是stack frame 的优化， 简单说来也就是 当 两个以上的函数同时调用时，会将上一级函数使用并且当前函数也使用的寄存器入栈，所以相当于有可能入栈的操作不止进入一次，如果入栈的位置是同一个的话，将会大大降低其位翻转。

  ```pseudocode
  //例子如下：
  
  
  int main() {
      f();
      g();
  }
  
  //如果该入栈为
  //如果主函数使用了 a，b，c，d寄存器,f使用了a、b、c，g使用了b、c、d。
  //普通的入栈
  f():
  push(a);
  push(b);
  push(c);
  ……
  pop(a);
  pop(b);
  pop(c);
  g():
  push(b);
  push(c);
  push(d);
  ……
  pop(b);
  pop(c);
  pop(d);
  
  //这样子最大的问题是相同寄存器明明要入栈两次，但是位置不一样，还是得需要位翻转，如果位置一样，就不用位翻转了
  
  //所以可以变成这样
  f():
  push(a);
  push(b);
  push(c);
  sp += 2;
  ……
  pop(a);
  pop(b);
  pop(c);
  sp -= 2;
  g():
  sp += 2
  push(b);
  push(c);
  push(d);
  ……
  pop(b);
  pop(c);
  pop(d);
  sp -= 2;
  ```

- memory molloc在分配不同大小的内存时，位翻转会有所不同。在内存对齐时（eg，16B,  32B，48B），起初会有一些差异，但是之后每字节的位翻转是相同的。在内存不对齐时（eg，40B），每字节的位翻转比对齐时的位翻转略微高。

#### 1.4、总的来说，作者的优化有一些是比较鸡肋的。有些实验的成功完全取决于作者设置实验的条件。但是作者提出的位翻转优化是可以进行探讨的一个大方向之一，应该可以从很多的存储结构中优化位翻转。

### 2、Rethinking File Mapping for Persistent Memory

#### 2.1、作者本文提出了对PM存储结构的文件映射系统的思考，提出了两种不同于传统文件映射的具有PM特色的Hash 文件映射结构。

#### 2.2、作者在文中提及的四种文件映射结构

- Extent trees， 包含范围的B树变形结构
- Redix trees， 采用基数来判断树的分支节点，类似于trie字典树的变形结构
- Global Cuckoo Hash Table，采用两次hash函数，当两个hash候选位置都满了时，采取移位方法，将其中的一个元素移动到t它的另一个候选上。
-  HashFS，仅仅采用最简单的hash算法实现，将整个表分为元数据区域和数据区域，元数据区域作为hash映射在数据区域上。通过链表来避免冲突，但是分配内存由单独设计的内存分配器来执行，分配数据区域的内存。

#### 2.3、最后实验表明了HashFS有最好的应用性能，尤其是在插入方面，比最好的PM文件映射性能提升了大概45%。