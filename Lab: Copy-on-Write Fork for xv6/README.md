# Lab: Copy-on-Write Fork for xv6

任务： 实现 `fork` 的 `lazy allocation`，也就是说 fork 时并不会真正的分配复制父进程的页表，而是与父进程指向相同的地址，直到某个进程需要更改时才正真的分配该页。

1. 新建一个内核中的数组，寄了每一页被引用的次数
```c
// 物理页到数组中的 ID    
#define PA2PGREF_ID(p) (((p)-KERNBASE)/PGSIZE)
// MAX_PAGE最大页数
#define MAX_PAGE PA2PGREF_ID(PHYSTOP)
// 锁与索引数组
struct spinlock pageReflock;
int pageRef[MAX_PAGE];
// 指向页对应位置的宏
#define PAGEREF(p) (pageRef[PA2PGREF_ID(p)])

// 初始化
void
kinit()
{
  initlock(&kmem.lock, "kmem");
  initlock(&pageReflock, "pageRef");
  freerange(end, (void*)PHYSTOP);
}

// 修改 kfree
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // 判断是否这个页面没有被引用了
  acquire(&pageReflock);
  if(--pageRef[PA2PGREF_ID((uint64)pa)] > 0) {
    release(&pageReflock);
    return;
  }
  release(&pageReflock);

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  acquire(&kmem.lock);
  r->next = kmem.freelist;
  kmem.freelist = r;
  release(&kmem.lock);
}
// 修改 kalloc
void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r)
    kmem.freelist = r->next;
  release(&kmem.lock);

  if(r) {
    memset((char*)r, 5, PGSIZE); // fill with junk
    //referance ++
    pageRef[PA2PGREF_ID((uint64)r)] = 1;
  }
  return (void*)r;
}

// 再添加两个函数
// 当引用大于一时，复制新页
void *kCOWcopy(void *pa) {
  acquire(&pageReflock);

  if(PAGEREF((uint64)pa) <= 1) { // 只有 1 个引用，无需复制
    release(&pageReflock);
    return pa;
  }

  // 分配新的内存页，并复制
  uint64 newpage = (uint64)kalloc();
  if(newpage == 0) {
    release(&pageReflock);
    return 0; // out of memory kill process
  }
  memmove((void*)newpage, (void*)pa, PGSIZE);

  // 旧页的引用减 1
  PAGEREF((uint64)pa)--;

  release(&pageReflock);
  return (void*)newpage;
}
// reference ++；
void pageRefAdd(uint64 pa){
  acquire(&pageReflock);
  ++PAGEREF(pa);
  release(&pageReflock);
}

// 添加到 def.h/kalloc.c
void            *kCOWcopy(void *);
void            pageRefAdd(uint64);
```

2. 观察 `fork` 的实现，发现其中复制父进程内存的函数是 `uvmcopy`，所以要修改该函数，实现与父进程指向同一地址，修改时才通过中断实现复制
```c
// riscv.h  中先定义个特殊的标志位
#define PTE_COW (1L << 8)// COW

// vm.h    COW uvmcopy 
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
  // char* mem

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    // 改为不可写且加上 COW 标签
    if(*pte & PTE_W) {
      *pte = (*pte & ~PTE_W) | PTE_COW;
    }
    flags = PTE_FLAGS(*pte);
    // 不用复制了
    // if((mem = kalloc()) == 0)
    //   goto err;
    // memmove(mem, (char*)pa, PGSIZE);
    // new 直接映射 与 old 相同的物理地址
    if(mappages(new, i, PGSIZE, (uint64)pa, flags) != 0)
      goto err;
    // 该物理页面的引用次数 ++
    pageRefAdd(pa);
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```
3. 处理 `page fault` 中断，`uvmIsCOWpage` 来判断这个页是不是指向的页是否是 `COW`, `uvmCOWcopy` 实现真实的页分配，如果页已经不足，此处`OOM` 策略为直接杀死进程。
```c
void
usertrap(void)
{
  // 省略
  if(r_scause() == 8){
    // system call
  } else if((which_dev = devintr()) != 0){
    // ok
  } else if((r_scause() == 13 || r_scause() == 15) && uvmIsCOWpage(r_stval())) { // copy-on-write
    if(uvmCOWcopy(r_stval()) == -1){ // 如果内存不足，则杀死进程
      p->killed = 1;
    }
  } else {
    
  }
  // 省略
}
```
4. 实现 `uvmIsCOWpage` 和 `uvmCOWcopy`，在次之前要实现两个查看当前进程栈指针和页表的函数 `getProc_sz` 和 `getProc_PT`
```c
// proc.h
uint64 getProc_sz(struct proc* p) {
  return p->sz;
}

pagetable_t getProc_PT(struct proc* p){
  return p->pagetable;
}
// def.h
uint64          getProc_sz(struct proc*);
pagetable_t     getProc_PT(struct proc*);

// vm.c
// 检查一个地址指向的页是否是 COW
int uvmIsCOWpage(uint64 va) {
  pte_t *pte;
  struct proc *p = myproc();
  // 判断合法、物理页是否存在、且是否为 COW
  if(va < getProc_sz(p) && ((pte = walk(getProc_PT(p), va, 0))!=0) && (*pte & PTE_V)&& (*pte & PTE_COW))
    return 1;
  return 0;
}

// 复制一个懒复制页，并重新映射为可写
int uvmCOWcopy(uint64 va) {
  pte_t *pte;
  struct proc *p = myproc();

  if((pte = walk(getProc_PT(p), va, 0)) == 0)
    panic("uvmCOWcopy: walk");
  
  // (如果懒复制页的引用已经为 1，则不需要重新分配和复制内存页，只需清除 PTE_COW 标记并标记 PTE_W 即可)
  uint64 pa = PTE2PA(*pte);
  uint64 new = (uint64)kCOWcopy((void*)pa); // 将一个懒复制的页引用变为一个实复制的页
  if(new == 0)
    return -1;
  
  // 重新映射为可写，并清除 PTE_COW 标记
  uint64 flags = (PTE_FLAGS(*pte) | PTE_W) & ~PTE_COW;
  uvmunmap(getProc_PT(p), PGROUNDDOWN(va), 1, 0);
  if(mappages(getProc_PT(p), va, 1, new, flags) == -1) {
    panic("uvmCOWcopy: mappages");
  }
  return 0;
}
```
5. 最后把测试 `cowtest` 加入 `Makefile` 即 

测试结果
```sh
// cowtest.c
$ cowtest
simple: ok
simple: ok
three: ok
three: ok
three: ok
file: ok
ALL COW TESTS PASSED
$ 

// make grade
== Test running cowtest == 
$ make qemu-gdb
(10.8s) 
== Test   simple == 
  simple: OK 
== Test   three == 
  three: OK 
== Test   file == 
  file: OK 
== Test usertests == 

$ usertests -q
...
ALL TESTS PASSED
```