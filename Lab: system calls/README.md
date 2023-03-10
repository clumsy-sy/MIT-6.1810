# Lab: system calls

In the last lab you used systems calls to write a few utilities. In this lab you will add some new system calls to xv6, which will help you understand how they work and will expose you to some of the internals of the xv6 kernel. You will add more system calls in later labs.

```sh
git checkout syscall
```

## System call tracing (moderate)

任务: 写一个 `trace` 系统调用, 跟踪输出程序运行中所用到的系统调用

1. 添加 `$U/_trace\` 到 `Makefile`
2. 在 `user/user.h` 中添加系统调用 `int trace(int)`; 在 `user/usys.pl` 中添加系统调用 `entry("trace");`; 在 `kernel/syscall.h` 中添加 `System call numbers` `#define SYS_trace  22`
4. 由于 `trace` 是需要跟踪进程的,所以在 `kernel/proc.h` 的 `proc` 结构体中添加
```c
struct proc{
    //...
    // trace
    int trace_mask;
}
```
5. 然后在 `kernel/sysproc.c` 写函数内获取`
```c
uint64
sys_trace(void)
{
  argint(0, &(myproc()->trace_mask));
  return 0;
}
```
6. 在 `kernel/proc.c` 内修改 `fork()`, 将 `trace` 传入子进程
```c
fork() {
  //...
  // copy trace
  np->trace_mask = p->trace_mask;
  //...
}
```
7. 在 `syscall` 中输出需要监测的系统调用, 并添加申明
```c
// add trace
extern uint64 sys_trace(void);
static uint64 (*syscalls[])(void) = {
    ...
    [SYS_trace]   sys_trace,
}
// 加一个数组,方便输出调用的名字
static char* syscalls_name[] = {
  "",
  "fork",
  "exit",
  "wait", 
  "pipe", 
  "read",
  "kill", 
  "exec", 
  "fstat",  
  "chdir",
  "dup",
  "getpid",
  "sbrk",
  "sleep",
  "uptime",
  "open",
  "write",
  "mknod",
  "unlink",
  "link",
  "mkdir",
  "close",
  "trace",
};
// 修改系统调用的函数
void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    // Use num to lookup the system call function for num, call it,
    // and store its return value in p->trapframe->a0
    p->trapframe->a0 = syscalls[num]();

    // trace the syscall
    if((1 << num) & p->trace_mask)
      printf("%d: syscall %s -> %d\n", p->pid, syscalls_name[num], p->trapframe->a0);
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

运行
```sh
init: starting sh
$ trace 32 grep hello README
3: syscall read -> 1023
3: syscall read -> 961
3: syscall read -> 321
3: syscall read -> 0
$ 
$ trace 2147483647 grep hello README
5: syscall trace -> 0
5: syscall exec -> 3
5: syscall open -> 3
5: syscall read -> 1023
5: syscall read -> 961
5: syscall read -> 321
5: syscall read -> 0
5: syscall close -> 0
$ 
$ grep hello README
$ 
$ trace 2 usertests forkforkfork
usertests starting
9: syscall fork -> 10
test forkforkfork: 9: syscall fork -> 11
11: syscall fork -> 12
12: syscall fork -> 13
12: syscall fork -> 14
13: syscall fork -> 15
12: syscall fork -> 16
13: syscall fork -> 17
15: syscall fork -> 18
12: syscall fork -> 19
13: syscall fork -> 20
14: syscall fork -> 21
12: syscall fork -> 22
13: syscall fork -> 23
15: syscall fork -> 24
19: syscall fork -> 25
14: syscall fork -> 26
16: syscall fork -> 27
12: syscall fork -> 28
18: syscall fork -> 29
13: syscall fork -> 30
17: syscall fork -> 31
19: syscall fork -> 32
12: syscall fork -> 33
13: syscall fork -> 34
14: syscall fork -> 35
15: syscall fork -> 36
12: syscall fork -> 37
13: syscall fork -> 38
15: syscall fork -> 39
14: syscall fork -> 40
13: syscall fork -> 41
15: syscall fork -> 42
14: syscall fork -> 43
13: syscall fork -> 44
15: syscall fork -> 45
14: syscall fork -> 46
12: syscall fork -> 47
13: syscall fork -> 48
14: syscall fork -> 49
16: syscall fork -> 50
13: syscall fork -> 51
14: syscall fork -> 52
12: syscall fork -> 53
13: syscall fork -> 54
15: syscall fork -> 55
14: syscall fork -> 56
12: syscall fork -> 57
15: syscall fork -> 58
13: syscall fork -> 59
51: syscall fork -> 60
12: syscall fork -> 61
14: syscall fork -> 62
15: syscall fork -> 63
12: syscall fork -> 64
14: syscall fork -> 65
13: syscall fork -> 66
15: syscall fork -> 67
17: syscall fork -> 68
13: syscall fork -> 69
12: syscall fork -> 70
15: syscall fork -> 71
14: syscall fork -> -1
13: syscall fork -> -1
12: syscall fork -> -1
15: syscall fork -> -1
OK
9: syscall fork -> 72
ALL TESTS PASSED
$ 
```

测试
```sh
make: 'kernel/kernel' is up to date.
== Test trace 32 grep == trace 32 grep: OK (1.0s) 
== Test trace all grep == trace all grep: OK (0.7s) 
== Test trace nothing == trace nothing: OK (1.1s) 
== Test trace children == trace children: OK (15.5s)
```

## Sysinfo (moderate)

任务: 写一个 `sysinfo` 系统调用, 可以获取 `free memory` 和处于 `UNUSED` 状态的程序的数量

1. 在 `Makefile` 中添加 `$U/_sysinfotest\`
2. 在 `user/user.h` 中添加系统调用
```c
struct sysinfo;
int sysinfo(struct sysinfo*);
```
3. 在 `usys.pl` 中添加 `entry("sysinfo");`
4. 在 `kernel/syscall.h` 中添加 `#define SYS_sysinfo 23`
5. 在 `kernel/syscall.c` 中添加
```c
extern uint64 sys_sysinfo(void);
static uint64 (*syscalls[])(void) = {
// ...
[SYS_sysinfo] sys_sysinfo
};
static char* syscalls_name[] = {
    // ...
    "sysinfo"
}
```
6. 在 `kernel/kalloc.c` 中加入计算剩余内存的函数
```c
// 获取 freemem 的函数
void
getFreeMemory(uint64 *freemem) {
  struct run *r;
  acquire(&kmem.lock);
  r = kmem.freelist;
  while(r) {
    r = r->next;
    *freemem += PGSIZE;
  }
  release(&kmem.lock);
}
```
7. 在 `kernel/proc.c` 中添加获取**不是** `UNUSED` 进程的个数
```c
void
getNumOfProc(uint64 *nproc) {
  struct proc *p;
  for(p = proc; p < &proc[NPROC]; p++){
    if(p->state != UNUSED)
      (*nproc) ++;
  }
}
```
8. 在 `kernel/sysfile.c` 内写系统调用 `sysinfo` 的实现
```c
void getFreeMemory(uint64*);
void getNumOfProc(uint64*);
uint64
sys_sysinfo(void) {
  struct sysinfo info = {0, 0};
  getFreeMemory(&info.freemem);
  getNumOfProc(&info.nproc);
  // 将数据从内核态带到用户态
  uint64 dstaddr;
  argaddr(0, &dstaddr);
  //判断 copyout 的返回值是必须的, sysinfotest内有一个测试点
  if(copyout(myproc()->pagetable, dstaddr, (char*)&info, sizeof info) < 0)
    return -1;
  return 0;
}
```

运行
```sh
init: starting sh
$ sysinfotest
sysinfotest: start
sysinfotest: OK
$ 
```

测试
```sh
❯ ./grade-lab-syscall sysinfo
make: 'kernel/kernel' is up to date.
== Test sysinfotest == sysinfotest: OK (3.3s)
```
