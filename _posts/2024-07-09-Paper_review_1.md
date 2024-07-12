---
layout: post #ensure this one stays like this
read_time: true # calculate and show read time based on number of words
show_date: true # show the date of the post
title:  Paper_review_SQL聚合和窗口函数
date:   2024-7-9 # XXXX-XX-XX XX:XX:XX XXXX
description: null
img:  posts/20230803/queue.jpg # the path for the hero image, from the image folder (if the image is directly on the image folder, just the filename is needed)
tags: [Database, SQL, Merge sort tree]
author: BreejojoDe
github: username/reponame/ # set this to show a github button on the post
toc: yes # leave empty or erase for no table of contents
---


**Window function**:  
窗口函数（Window Functions）是 SQL 中的一种高级分析工具，用于在一个查询中计算跨多行的数据聚合和分析。与传统的聚合函数（如 SUM、COUNT、AVG 等）不同，窗口函数不会将多行数据合并成一行，而是保留每行数据，并在此基础上计算额外的结果。  
就像在行上套了个窗口，把窗口内的数据聚合分析后结果加在一个行里  


1. 单线程。数据库中无法使用
2. $(O^2)$ 复杂度  


**OLAP Query**  
多维数据分析


## 2 Window Function in SQL

### 2.1 汇总功能
- distributive aggregates (COUNT, SUM, . . .), 与输入线性关系
- Algebraic aggregates (AVG, . . .)， 由distributive组合
- Holistic aggregates，线性扫描解决不了的

### 2.2 语法
```sql
function(arguments) over (partition by ...
    order by ... rows between ...)
```
- partition by 分区，每个区之间数据相互隔离
- order by 分区内的顺序-相邻
- 选择邻居的子集  *window frame*
    - rows 行偏移
    - range 值范围
    - `EXCLUDE CURRENT ROW, EXCLUDE TIES, . . .` 排除 
-   


## 3 相关工作

### 3.1 研究领域

**线段树 segment trees**：
线段树（Segment Tree）是一种平衡二叉树，用于高效处理区间查询和更新操作。被广泛使用在需要频繁查询和修改数据的情况下。  
构建 $O(n)$, 查询 $O(log n)$  
平衡二叉树实现，
- 叶节点：存储数组的原始元素。
- 内部节点：存储其子树包含的数据的聚合信息（如和、最小值、最大值等），而不存储原始数据。  
![alt text](./assets/img/posts/20240709/image-1.png)

*Q：线段树和B+树在索引上有什么区别*
<br>  
**增量算法 incremental algorithms**：增量算法（Incremental Algorithms）是一类特殊的算法，用于在数据发生变化时高效地更新计算结果，而不是每次都重新计算。增量算法通过利用之前计算的部分结果，只对变化部分进行更新，从而显著提高效率。
<br>  
**顺序统计树 Order Statistic Tree**：在平衡二叉树上加了一个size用于记录包括自身在内的所有子树的size，可以在二叉树上直接实现选择第k位和排名。复杂度都是$logn$，整体排名就是$nlogn$。  
所以这里是依赖子树才能确定size，操作需要串行化，不能并行
<br>  
**归并排序树Merge Sort Tree**：在归并的过程中构建树，先把数据分割为节点，然后从最底层逐层排序，排序结果储存在上一层里，每个节点都储存了子树的排序结果，空间$O(nlogn)$。  
- MST可以并行化的基础是 
    1. 对节点的分割，即叶节点储存输入数据，这个过程是可以并行化的；
    2. 每层节点之间的处理不受相互影响，完全可以并行化处理达到 $logn$  
<br>

**范围树 Range Tree**：指定多维的某一维为基准构建平衡二叉树，选取中位数为根，左右递归。  
<br>
**分数级联 Fractional Cascading**：  
引入哨兵值来对多段数据进行排序，可以从一个数据段德哨兵值和指针跳转到另一个数据段的对应位置，理论上可以将k*n的复杂度变为k+n。  

![alt text](./assets/img/posts/20240709/image.png)  

### 3.2 评估

- MST结构上支持并行化
- 对比增量算法，MST用更多的空间，但用了更少的时间，时间是限制distinct和percentiles的主要因素。
- 复杂聚合操作需要维护一个所有窗口共享的聚合状态，有的计算需要某一任务调用其他任务的结果，这就无法分配任务到不同并行线程，强行分配会有很多重复任务，n次n个元素，变成$n^2$复杂度
- 边界非单调的窗口
  
#### 对于分段树
- 并行化处理分布式和代数聚合（查询复杂度$O(nlogn)$）
- 整体聚合：查询时，每个节点的合并操作复杂度为$O(logn)$，因为每个节点存储的是有序列表。由于需要合并$O(logn)$个节点，整体查询复杂度为$O((logn)^2)$。问题焦点在于无法在O(1)的时间内归并为聚合状态。
  
#### 对于MST
- 可以完成多种聚合操作，则无需专门设计多种
- 并行合并的操作依赖merge，可以直接利用现有操作  

## 4 MST Holistic Aggregates

### 4.1 概览

### 4.2 窗口化 COUNT DISTINCT 操作

构建一个前置位置标记的数组，储存对应的元素的上一次出现的index，如图所示
![alt text](./assets/img/posts/20240709/image-2.png)  

对于窗口内的元素的前置索引，有两种情况，
- 在窗口内，证明这个元素在窗口内出现过，是重复的
- 不在窗口内，证明是窗口内第一次出现  

论文的算法实现：
![alt text](./assets/img/posts/20240709/image-3.png)  
*Q：这里返回的prevIdes的顺序是sorted数组排序之后的顺序，这个顺序与之前原数据的顺序并不一致，也就是说，in[i]的前置标记不是prevIdes[i]，是否需要对算法返回的prevIdes重新按sorted.secend排序一次？*  

直接使用这个方法的问题就是每次需要吧窗口内的元素一一对比，复杂度为n，要对比所有窗口，时间复杂度$O(n^2)$，因此进行改进。  

不直接检查，而是排序后检查，但是要保留preIdes的原顺序，所以采用MST的结构，留存中间每段的归并结果，这样可以根据区间找到已有的归并段，归并段是有序的，不需要一一对比，段内使用二分查找，这样就把层内的查找时间降低为$log n$，但注意这只是单层的。   
![alt text](./assets/img/posts/20240709/image-4.png)  

这时的查找还只是加入MST，我们在应用时还需要对查找段分割，这是一个查找“归并段”的过程，最暴力的就是逐层扫描，一共 log n 层，得到的单次窗口查询的复杂度是$O((logn)^2)$，则总的复杂度就是$O(n(logn)^2)$。这是查询的复杂度，构建的复杂度是$O(nlogn)$，所以需要继续优化查询过程。  

在对一层进行扫描时，如果进入的不是最终需要的归并段，那么就进入扫描中断处对应的下一层的扫描段进行扫描，或者说只在第一层扫描但是找的对应归并段需要一个flag指引，这个flag可以为每个元素拥有，它指向他在下一级的各个归并段最贴近的位置，这就是加入Fractional Cascading的归并树  
![alt text](./assets/img/posts/20240709/image-5.png)  
![alt text](./assets/img/posts/20240709/image-6.png)  
*Q in reading：这里并没有给出一个明确的实施方法，如何构建这个flag，就是文里所说的注释，以及如何查找，但所说的办法逻辑上确实成立*  


### 4.3 SUM DISTINCT
维护一个“注释”，对于每个元素i，注释就是在该归并段内的前i个元素的和，因为归并段有序，所以第i个元素及之前都是小于等于i元素值，第i个元素之后都是大于i元素值，可以做到选取前i个不重复的值的和这一操作。
![alt text](./assets/img/posts/20240709/image-7.png)  

使用 Fractional Cascading 使找到所有归并段花费$logn$，而最多需要合并归并段$logn$，则单个窗口结果为$O(logn)$，于是总共需要$O(nlogn)$。  


### 4.4 Windowed Rank  
  
ORDER BY：
- 函数内
- 窗口内
  
Rank, Percent_Rank可以使用MST实现，本质上变成计算小于当前行的元组数，可以用MST搜索，单个操作$log n$，
  
ROW_NUMBER, CUME_DIST：用位置区分各个元素  

DENSE_RANK无法使用，会导致三维查询，需要用范围树  


### 4.5 Percentiles and Value Functions  

百分数和值函数都是选择第i个最小值，相当于查询窗口里的某个具体位置的值，最原始的思维是排序后选择第i个，至于是否在窗口里，通过排序时保留位置信息，位置在窗口内就统计。  
  
这样的方法获得的是遍历排序后的数组才能获得结果，复杂度$O(n)$，整体聚合就是$O(n^2)$  
![alt text](./assets/img/posts/20240709/image-8.png)  
  
改进是用MST来搜索，把排序后的位置索引作为MST的最底层，然后构建MST，从上向下搜索，搜索到最底层的时候就能获得真实排名，搜索原则在论文中是这样：**In case the number of qualifying elements in the left sublist is smaller than the searched element, we descend to the left,
otherwise to the right.** 这里的searched element意义难以明确，但思维可以理解。  
思想就是查找每个部分的的左归并段，如果找到的数量大于等于i，说明就在left里，否则，在right里；同时每向下一层就可以实时更新这个 $i$，如果查找left，i不变，查找right，i就变，这样查找到最底下就是我们要的数据的index。具体如图。  
![alt text](./assets/img/posts/20240709/image-9.png)  

不同于COUNT只有一个边界（因为COUNT构造的前置索引数组的单调性），这里的排序后的索引归并段，需要一前一后两个边界（即查找的窗口的前后界），于是对每一层我们查找的有序的归并段，都需要两次二分查找，这样就需要单层的$O(logn)$，一个树的搜索就是$O((logn)^2)$，这样得到的整体聚合复杂度还是$O(n(logn)^2)$。  

注意到问题与前一个MST相同，都是不同层之间重复使用二分查找，所以可以与其解决办法类似，使用  fractional cascading，同理，复杂度会降到$O(n(logn))$。  

对于NULL值的处理，VALUE FUNC和PERCENT都有忽略NULL值的情况，只需要在忽略NULL值的数据上建立MST即可。  

这个整体聚合的步骤如下：  
1. Compute the permutation array → $O (𝑛 log𝑛)$，这里的“置换数组”就是以value为关键字的排序，值就是每个value的索引位置。
2. Build the merge sort tree → $O (𝑛 log𝑛)$  
3. Compute the percentile for all 𝑛 input tuples → $O (𝑛 log𝑛)$  


### 4.6 LEAD & LAG

窗口这个操作实现可以用带有两个独立Order By这个框架，  

- LEAD：用于访问CURRENT_ROW之后的行数据
- LAG：用于访问CURRENT_ROW之前的行数据

这两个语句都会提供偏移量，并可能有ORDER BY，因此我们需要获得在某一排序下当前行的偏移行，我们可以先使用4.5的MST对数据格式化为MST，然后用ROW_NUMBER的实现方法实现行号唯一标识，获取当前行号后计算偏移后的行号，再用4.5的方式搜索第i个，得到的就是偏移行。  
*事实上，当搜索范围扩大到整个归并段时，我们可以设置特殊的优化（待思考）*


### 4.7 非连续window

可以视为由常数个连续的window组合，所以时间复杂度的渐进曲线量级不变。  
在实现过程中，可以在构建MST时，将被选中的元组作为底层数据，即剔除没被选中的数据，再编写合适的规则映射索引。  

*ps.听起来挺合理的，但具体实现还需考量*  


## 5 实现的细节论述  

### 5.1 内存
MST置于内存（可磁盘），每级使用一个数组保存  

对RANK的初始排序数组的储存，储存其排序顺序，如图  
![alt text](./assets/img/posts/20240709/image-10.png)  
  
根据输入数据大小动态调整储存数据类型  

对于fractional cascading的实现，层与层之间的指针只在“第k个”元素上，这是一种常数的抽样方式，k值的选择会影响内存带宽和内存消耗之间的平衡。  

论文里涉及到的“高扇出f”应该指MST的每个节点最大子树的个数，即从二路归并拓展到f路归并，这样每层的二分查找数量会增加，但是会有更低的内存开销和更好的局部性。  
经验得到 $f = k = 32$  

### 5.2 并行化

#### 并行预处理
parallel_for  

#### 并行排序  
每个任务使用 introsort 对数据的一部分进行排序。其中，introsort 是一种混合排序算法，结合了快速排序和堆排序的优点。

#### 并行合并
使用合适的分割方法把每个runs分割，分割后把不同runs同位置的片段放在一个任务里并行，这样就实现了简单的合并任务的并行。论文中采用的方法是用中位数划分，要找到所有中位数的中位数。  

#### 构建MST
- 每个任务独立合并低level
- 多个线程协作合并高level  
    *线程协作需要编程实现，这里只是概念上的解释*

### 5.3 Code Reuse  
可重用数据库的并行排序代码  

introsort实现的quicksort部分是双向局部排序，为了优化有较多重复值的情况，改进为3路分区快排  
```cpp
void quicksort(std::vector<int>& arr, int low, int high) {
    if (low < high) {
        int lt = low, gt = high;
        int pivot = arr[low];  // 基准值
        int i = low;

        while (i <= gt) {
            if (arr[i] < pivot) std::swap(arr[lt++], arr[i++]);
            else if (arr[i] > pivot) std::swap(arr[i], arr[gt--]);
            else i++;
        }

        quicksort(arr, low, lt - 1);
        quicksort(arr, gt + 1, high);
    }
}

```

### 5.4 Code Generation  


## 6 评估  

- 与帧大小无关，50000时超过OST成为最高
- 并行优势
- 非单调的边界下的优势
- 带来额外的内存开销（MST的储存、FC的采样）


## 7 结论

- 支持现有SQL功能的window func整合
- 在关系数据库实现上提供$O(n(logn))$的思路
- 在实现上支持并行化，发挥多核计算单元的作用
- 为流聚合系统提供参考