# lab3 Report

2016011398 高鸿鹏

#### 练习1：给未被映射的地址映射上物理页（需要编程）

##### 相关知识点整理如下：

```
pte_t *get_pte(pgdir,la,create)	得到一个页表项并返回页表项对应的逻辑地址，如果页表项对应的页面未分配，则分配一个页面(页项第三个bit表示对应的页面是否存在)

pgdir_alloc_page(pgdir,la,perm)	调用alloc_page和page_insert来分配一页的内存，并且建立pgdir和pt的映射，la和pa的映射,perm表示【此页的权限在相关的pte中设置】

VM_WRITE						vma_flags & VM_WRITE = 1/0表示vma 可写/不可写
PTE_W							可写
PTE_U							用户可访问
mm->pgdir						vma对应的页目录表
```

##### 根据代码注释整理流程：

```
1. 寻找页目录项，如果页目录项对应的页表不存在，则创造一个页表
2. 如果物理地址不存在，分配一个物理页，建立pa和la之间的映射
```

##### 代码实现如下：

```c
	ptep = get_pte(mm->pgdir,addr,1);

    if(ptep == NULL){
        cprintf("get_pte failed in function do_pgfault\n");
        goto failed;
    }

    if(*ptep == 0){
        struct Page *page = pgdir_alloc_page(mm->pgdir,addr,perm);
        if(page == NULL){
            cprintf("pgdir_alloc_page falied in function do_pgdefault\n");
            goto failed;
        }
    }
```

##### 和答案比较：

```
1. 未考虑ptep == NULL和page == NULL的边界情况，参考答案后补齐
```

- ##### 请描述页目录项（Page Directory Entry）和页表项（Page Table Entry）中组成部分对ucore实现页替换算法的潜在用处。

  页目录项如图：

  ```
   31               12 11 9 8 7 6 5 4 3 2 1 0
  +-------------------+----+-+-+-+-+-+-+-+-+-+
  |   页表基址高20位   |忽略|G|S|0|A|C|T|U|W|P|
  +-------------------+----+-+-+-+-+-+-+-+-+-+
  ```

  1. A表示在上次清零之后是否被访问过,可以用于时钟替换算法
  2. D表示上次清零之后是否被写过，可以用于改进的时钟替换算法

  页表项如图：

  ```
   31               12 11 9 8 7 6 5 4 3 2 1 0
  +-------------------+----+-+-+-+-+-+-+-+-+-+
  |   页表基址高20位   |忽略|G|S|0|A|C|T|U|W|P|
  +-------------------+----+-+-+-+-+-+-+-+-+-+
  ```

  1. P用来表示页面是否在内存中，能够存放交换分区相关的信息，在所有的页免提换算法中都能用得到

- ##### 如果ucore的缺页服务例程在执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情

  	1. 处理器会将异常访问的地址存储在CR2寄存器中
  	2. 查找IPT找到中断服务程序的入口
  	3. 操作系统向当前栈压入EFLAGS，CS，EIP，errorCode信息，进入中断服务例程

#### 练习2：补充完成基于FIFO的页面替换算法（需要编程）

##### `do_pgdefault`相关知识点整理：

```
swap_in(mm,addr,&page)						分配一页内存，根据页表项中需要交换的地址，找到需要交换的磁盘页，将磁盘中的页读到内存中

page_insert(pgdir,page,la,perm)				建立la，pa之间的映射

swap_map_swappable(mm,addr,page,swap_in)	设置页面可交换
```

##### `do_pgdefault`根据代码注释整理流程：

```c
if(init_swap_ok){
    1. 根据mm,addr，将正确的磁盘页面内容加载到页管理的内存中
    2. 根据mm,addr,page完善la和pa之间的映射
    3. 设置页面可交换
}else{
    cprintf("no swap_init_ok but ptep is %x, failed\n",*ptep);
    goto failed;
}
```

##### 根据代码注释补全`do_pgdefault`：

```c
else{
        if(swap_init_ok){
            struct Page *page = NULL;
            int ret_code = swap_in(mm,addr,&page);

            if(ret_code != 0){
                cprintf("swap_in falied in function do_pgdefault\n");
                goto failed;
            }
            page_insert(mm->pgdir,page,addr,perm);
			swap_map_swappable(mm,addr,page,1);
			page->pra_vaddr = addr;
        }else{
            cprintf("no swap_init_ok but ptep is %x, failed\n",*ptep);
            goto failed;
        }
    }
```

##### 和参考答案对比后发现

```
1. 忽略了ret_code不为0的情况
2. 缺少对page->pra_vaddr属性的设置
```

##### 相关知识点整理`swap_fifo.c`

```
为了实现FIFO算法，需要将所有可交换页面链入pra_list_head，链表结构为双向链表，需要熟悉将链表结构转化为特殊链表结构(struct Page)，可能会用到le2page，le2vma，le2proc等

1. _fifo_init_mm:	初始化pra_list_head，让mm->priv指向pra_list_head所在的地址

2. _fifl_map_swappable:	根据FIFO算法将最近访问过的页面加入pra_list_head的最后
```

根据代码注释完成实验即可

##### 和参考答案相比

```
1. 没有考虑环状链表只有一个节点的情况
2. 没有考虑到链表节点和页映射失败的情况
```

##### 思考题

```
如果要在ucore上实现"extended clock页替换算法"请给你的设计方案，现有的swap_manager框架是否足以支持在ucore中实现此算法？如果是，请给你的设计方案。如果不是，请给出你的新的扩展和基此扩展的设计方案。并需要回答如下问题
需要被换出的页的特征是什么？
在ucore中如何判断具有这样特征的页？
何时进行换入和换出操作？
```

现有框架可以实现此算法，设置一个全局页指针作为时钟指针，每次插入直接插入到head前即可，在换出时，需要进行判断访问位A，写入位D的取值对（A,D），进行不同的操作。

```
(0,0) 					换出并让指针指向下一页。
(0,1) -> (0,0)			需要将该页写入交换区，指针指向下一页继续查找。
(1,1) -> (1,0)			指针指向下一位继续查找。
```

1. 需要被换出的页的特征是什么？

   近期没有被访问过和修改过的页面

2. 在ucore中如何判断具有这样特征的页？

   只要能找到访问位和写入位均为0的页面即可

3. 何时进行换入和换出操作？

   当前访问的虚拟页没有物理页对应的时候