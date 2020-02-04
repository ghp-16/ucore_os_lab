# lab1 Report

2016011398 高鸿鹏

#### 练习1：实现 first-fit 连续物理内存分配算法（需要编程）

相关知识点整理如下：

```
free_area_t结构被用于分配空闲内存块

首先需要看list.h中来了解链表结构，为简单的双向链表，熟练使用list_init,list_add,list_add_before/after,list_del和list_next,list_pre等函数

可以利用函数进行特别的结构转换，如le2page，le2ma，le2proc等

default_init：重写该函数来初始化free_list，将nr_free置为0，free_list用来记录空闲内存块，nr_free时是空闲内存块的数量

default_init_memmap：用于初始化一个空闲内存块(参数为address_base,page_number)，首先需要初始化每一个page(memlayout.h)，初始化包括以下步骤：

1. 设置p->flags的PG_property,表示当前page状态为valid，如果page空闲并且不处于空闲内存块的第一页，置0，如果page空闲并且处于空闲内存块的第一页，则置为当前内存块的总页数
2. p->ref应当置为0，因为当前页p是空闲的而且未被引用
3. 初始化之后我们使用p->page_link将当前页加入free_list   e.g.: `list_add_before(&free_list, &(p->page_link));`
4. 上述过程完成后`nr_free += n`更新空闲内存块总数

default_alloc_page：寻找free_list中第一个空闲内存块(block_size > n)，重置内存块大小，返回`malloc`要求的当前块的地址

1. 遍历free_list的方式如下

   list_entry_t le = &free_list;
   while((le=list_next(le)) != &free_list) {

   1.1 在while循环中，得到`page`结构并且检查`p->property>=n`

     struct Page *p = le2page(le, page_link);
     if(p->property >= n){

   1.2 如果能够找到这样的p，其前n个页就会被分配，需要更改对应的页的flags，`PG_reserved = 1`, `PG_property = 0`。然后从free_list中删除这些页

     1.2.1 如果p->property > n，重新计算空闲块的大小：

       le2page(le,page_link))->property = p->property - n;

   1.3 重新计算nr_free

   1.4 return   `p`

2. 如果找不到满足条件的块就 return NULL

default_free_page：将释放的page重连进free_list，可能需要合并小的空闲块

	2.1 根据基址，寻找合适的插入位置，可能会用到(list_next,le2page,list_add_before等函数)
	2.2 重设归还页的相关属性，p->ref,p->flags等
	2.3 尝试合并块，需要注意设置p->property
```



##### 1. 对`default_init_memmap`的修改

```
按照注释，将原有的list_add修改为list_add_before
```



##### 2. 对`default_alloc_page`的修改：

```
在找到合适的空闲页后，需要通过 SetPageProperty(p) 设置property为0，然后将新的空闲页加在找到的空闲页的后面，并将之前找到的合适的空闲页用从free_list中移除
```

##### 对default_free_page的修改：

按照注释发现当前free_page函数缺少了将合并完成后的空闲块加入free_list的过程，所以在后面加入寻找合适的插入位置的过程，待插入块基址+带插入块大小 <= 下一个空闲块的起始地址即可，然后将待插入块插入在目标空闲块的前面即可。

##### 可能的优化

​	在释放时的合并操作，在查找待插入块的下一个空闲块的位置时可以通过当前块的页面数得到，不需要使用循环。在分配的时候考虑cache，尽量分配cache中的页面，能够加快查找速度。

#### 练习2：实现寻找虚拟地址对应的页表项（需要编程）

根据注释得到相关函数的用途如下：

```
KADDR(pa)							用于访问物理地址，返回值为虚拟地址
PDX(la)								虚拟地址la对应的页目录表中的index
set_page_ref(page,1)				表示page的引用次数为1
page2pa(page)						得到页面对应的的物理地址
struct Page* alloc_page()			分配一个空白页
memset(void *s, char c, size_t n)	将s区域的前n个bit置为c
```

根据代码注释按顺序完成：

1. 找到页目录表项
2. 如果页目录表项对应的物理地址未分配，则检查是否需要分配物理页面
3. 如果需要分配物理页面，则分配新的物理页面，并设置当前页面的引用为1
4. 得到当前页面的物理地址，并使用memset清理页面内容，设置表项permission

##### 和答案比较

1. 和答案相比，我缺少了在alloc_page之后检测页面是否为空的情况

2. 设置页表项permission的时候顺序出现问题

##### 修改代码如下：

```
pde_t *pdep = &pgdir[PDX(la)];     //try to get page,failed into if
    if(!(*pdep & PTE_P)){
        struct Page *page;
        if(!create || (page = alloc_page()) == NULL){
            return NULL;        
        }
        //page = alloc_page();
        set_page_ref(page,1);
        uintptr_t pa = page2pa(page);
        memset(KADDR(pa),0,PGSIZE);
        *pdep = pa | PTE_U | PTE_W | PTE_P;
    }
    return &((pte_t *)KADDR(PDE_ADDR(*pdep)))[PTX(la)];
```

+ ##### 请描述页目录项（Page Directory Entry）和页表项（Page Table Entry）中每个组成部分的含义以及对ucore而言的潜在用处。

  - 页目录项表示如图（参考谭院士的报告找到的结构）：

```
     31               12 11 9 8 7 6 5 4 3 2 1 0
    +-------------------+----+-+-+-+-+-+-+-+-+-+
    |   页表基址高20位   |忽略|G|S|0|A|C|T|U|W|P|
    +-------------------+----+-+-+-+-+-+-+-+-+-+
```

    1. 其中基址为页表的物理地址，因为页表大小为4K，Offset为12位，所以表示高20位。
    2. G在页目录中没有具体作用
    3. S表示页大小，1为4M，0为4K值，在ucore中为0
    4. A表示在上次清零之后是否被访问过
    5. C置1表示缓存，否则不是
    6. T置1缓存写回，否则不是
    7. U置0表示页表中任何页面只能在内核态访问
    8. W表示页面是否可写，如果CR0的WP为0则不能写，内核态总可以写
    9. P表示页面是否存在于内存之中

  - 页表项表示如图

    ```
     31               12 11 9 8 7 6 5 4 3 2 1 0
    +-------------------+----+-+-+-+-+-+-+-+-+-+
    |   页基地址高20位   |忽略|G|0|D|A|C|T|U|W|P|
    +-------------------+----+-+-+-+-+-+-+-+-+-+
    ```

    1. 页面物理地址，因为页表大小为4K，Offset为12位，所以表示高20位。
    2. G表示这一页表项是否为全局项，若被置位，在CR3改变时，相应TLB不被清除，对于内核的页面映射，可以设置此位来优化内核访问内存的性能
    3. D表示在上次清零之后是否被写过
    4. A表示在上次清零之后是否被访问过
    5. C置1表示缓存，否则不是
    6. T置1缓存写回，否则不是
    7. U置0表示页表中任何页面只能在内核态访问
    8. W表示页面是否可写
    9. P表示页面是否存在于内存之中

+ ##### 如果ucore执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

  - 硬件陷入内核，在内核堆栈中保存程序计数器
  - 启动一个汇编代码例程保存通用寄存器和其他易失的信息
  - 硬件寄存器中保存需要的虚拟页面信息，即缺页中断的虚拟地址
  - 在发现选择的空闲页框“脏”的时候安排写回磁盘，发生一次上下文切换，挂起产生缺页中断的进程，让其他进程运行至磁盘传输结束

#### 练习3：释放某虚地址所在的页并取消对应二级页表项的映射（需要编程）

根据注释得到相关函数的用途如下：

```
struct Page *page pte2page(*ptep)	根据ptep的值找到对应的page
free_page(page)							释放page
page_ref_dec(page)					减少页面的引用次数，如果page->ref == 0,需要free_page
tlb_invalidate(pde_t *pgdir, uintptr_t la)	消除TLB中的一项(仅当正在编辑的页表是处理器当前正在												使用的页表时才使无效)
```

根据代码注释按顺序完成：

1. 检查ptep是否存在
2. 找到ptep对应的页面
3. 将ptep对应的页面的引用-1，当引用为0的时候释放页面
4. 清理ptep
5. 应用函数`tlb_invalidate`冲刷TLB

##### 修改代码如下：

```c
    if(*ptep & PTE_P){
        struct Page *page = pte2page(*ptep);
        page_ref_dec(page);
        if(page->ref == 0){
            free_page(page);        
        }
        *ptep = 0;
        tlb_invalidate(pgdir,la);
    }
```

- ##### 数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？

  Page结构体与物理页面一一对应，`pte2page`函数从页表项中找到对应的管理物理页表的page，`struct Page`与物理页表的起始地址一一对应。

- ##### 如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事？

  首先更改`KERNBASE`为`0x00000000`，原因在于在没有开启页式地址转换前,内核逻辑地址经过一次段地址转换即直接得到物理地址：PA(物理) = LA(线性) = VA(虚拟) - KERNBASE

  使用id工具形成的ucore起始虚拟地址从0xC0100000改成0x00100000