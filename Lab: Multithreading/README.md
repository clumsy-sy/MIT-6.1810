# Lab: Multithreading

## Uthread: switching between threads (moderate)

任务：完成用户态的“线程”

1. 保存 callee-saved 寄存器
```c
// uthread.c
struct context {
  uint64 ra;
  uint64 sp;
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};

struct thread {
  char       stack[STACK_SIZE]; /* the thread's stack */
  int        state;             /* FREE, RUNNING, RUNNABLE */
  struct context ctx;           /* Context save*/
};
```
2. 完成 `uthread_swithch.S` 内容和 `kernel/swtch.S` 一样
```asm
	.text

	/*
         * save the old thread's registers,
         * restore the new thread's registers.
         */

	.globl thread_switch
thread_switch:
	/* YOUR CODE HERE */
        sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)
	ret    /* return to ra */
```
3. 完善 `thread_schedule` 函数
```c
void 
thread_schedule(void)
{
  // ...
  // 线程切换
  if (current_thread != next_thread) {         /* switch threads?  */
    next_thread->state = RUNNING;
    t = current_thread;
    current_thread = next_thread;
    /* YOUR CODE HERE
     * Invoke thread_switch to switch from t to next_thread:
     * thread_switch(??, ??);
     */
    thread_switch((uint64)&t->ctx, (uint64)&next_thread->ctx);
  } else
    next_thread = 0;
}
```
4. 完善 `thread_create` 函数
```c
void 
thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }
  t->state = RUNNABLE;
  // YOUR CODE HERE
  t->ctx.ra = (uint64)func;       // 返回地址
  t->ctx.sp = (uint64)&t->stack + (STACK_SIZE - 1);  // 栈指针
}
```

运行
```sh
init: starting sh
$ uthread
thread_a started
thread_b started
thread_c started
thread_c 0
thread_a 0
thread_b 0
thread_c 1
thread_a 1
# ...
thread_a 99
thread_b 99
thread_c: exit after 100
thread_a: exit after 100
thread_b: exit after 100
thread_schedule: no runnable threads
$
```

测试
```sh
❯ ./grade-lab-thread uthread
make: 'kernel/kernel' is up to date.
== Test uthread == uthread: OK (2.4s) 
```

## Using threads (moderate)

运行源代码

```sh
## 编译
make ph
## 1
❯ ./ph 1                    
100000 puts, 6.246 seconds, 16011 puts/second
0: 0 keys missing
100000 gets, 6.266 seconds, 15960 gets/second
## 2
❯ ./ph 2
100000 puts, 2.781 seconds, 35959 puts/second
1: 16428 keys missing
0: 16428 keys missing
200000 gets, 6.729 seconds, 29724 gets/second
```

明显 2 的速度快但是丢失了很多数据，所以是线程不安全的，要加锁。

1. 初始化
```c
pthread_mutex_t locks[NBUCKET];

int
main(int argc, char *argv[])
{
  // ...

  // locks init
  for(int i = 0; i < NBUCKET; ++i) {
      pthread_mutex_init(&locks[i], NULL);
  }
  //
  // first the puts
  // ...
}
```

2. `insert` 枷锁
```c
static 
void put(int key, int value)
{
  int i = key % NBUCKET;

  // is the key already present?
  struct entry *e = 0;
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key)
      break;
  }
  if(e){
    // update the existing key.
    e->value = value;
  } else {
    pthread_mutex_lock(&locks[i]);
    // the new is new.
    insert(key, value, &table[i], table[i]);
    pthread_mutex_unlock(&locks[i]);
  }
}
```

运行

```sh
❯ ./ph 2 
100000 puts, 3.073 seconds, 32537 puts/second
0: 0 keys missing
1: 0 keys missing
200000 gets, 6.104 seconds, 32763 gets/second
```

测试
```sh
❯ ./grade-lab-thread ph_fast
make: 'kernel/kernel' is up to date.
== Test ph_fast == make: 'ph' is up to date.
ph_fast: OK (22.1s) 
```

## Barrier(moderate)

判断是不是所有线程都进入 `barrier` 函数内
```c
static void 
barrier()
{
  // YOUR CODE HERE
  //
  // Block until all threads have called barrier() and
  // then increment bstate.round.
  //
  pthread_mutex_lock(&bstate.barrier_mutex);
  if(++bstate.nthread < nthread) {
    pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
  } else {
    bstate.nthread = 0;
    bstate.round++;
    pthread_cond_broadcast(&bstate.barrier_cond);
  }
  pthread_mutex_unlock(&bstate.barrier_mutex);
}
```

运行

```sh
make barrier
# 2
❯ ./barrier 2               
OK; passed
# 5
❯ ./barrier 5
OK; passed
# 10
❯ ./barrier 10
OK; passed
```

测试
```sh
❯ ./grade-lab-thread barrier
make: 'kernel/kernel' is up to date.
== Test barrier == make: 'barrier' is up to date.
barrier: OK (3.4s) 
```