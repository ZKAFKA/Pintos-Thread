# 斯坦福大学操作系统课程设计 Pintos  
##### thread部分的完成
### Pintos的安装和配置
1. 安装qemu  
`sudo apt-get install qemu`  
2. 从git公共库获得最新Pintos  
`git clone git://pintos-os.org/pintos-anon`
3. /utils/pintos-gdb用vim打开，编辑GDBMACROS变量，将Pintos完整路径赋给该变量。
4. 用vim打开Makefile并将LOADLIBES变量名编辑为LDLIBS
5. 在/src/utils中输入make来编译utils
6. 编辑/src/threads/Make.vars（第7行）：更改bochs为qemu
7. 在/src/threads并运行来编译线程目录make
8. 编辑/utils/pintos（第103行）：替换bochs为qemu  
    编辑/utils/pintos（〜257行）：替换kernel.bin为完整路径的kernel.bin   	 
    编辑/utils/pintos（〜621行）：替换qemu为qemu-system-x86_64   
    编辑/utils/Pintos.pm（362行）：替换loader.bin为完整路径的loader.bin   
9. 打开~/.bashrc并添加export PATH=/home/.../pintos/src/utils:$PATH到最后一行。
10. 重新打开终端输入source ~/.bashrc并运行
11. 在Pintos下打开终端输入pintos run alarm-multiple

### Threads
#### Mission1
该部分内容主要是需要我们修改timer_sleep函数  
首先查看原本timer_sleep代码  
```
/* Sleeps for approximately TICKS timer ticks.  Interrupts must
   be turned on. */  
void  
timer_sleep (int64_t ticks)  
{  
  int64_t start = timer_ticks ();
  ASSERT (intr_get_level () == INTR_ON);
  while (timer_elapsed (start) < ticks)
    thread_yield();
}
```
通过查看相关教程及分析，可知start获取了起始时间， 然后断言必须可以被中断， 不然会一直死循环下去。
```
while (timer_elapsed (start) < ticks)
   thread_yield();
```
追踪thread_yield()函数及schedule()函数，schedule负责切换下一个线程进行run，thread_yield负责把当前线程放到就绪队列里，然后重新schedule（当ready队列为空时线程会继续在cpu执行）。  而timer_sleep则是在ticks时间内，如果线程出于running状态就不断把线程放到就绪队列不让他执行。由此可以发现它存在的问题：  
**线程不断在CPU就绪队列和running队列之间来回，占用了CPU资源。因此我们决定添加唤醒机制。**  
##### 函数实现
修改thread结构体：
```
struct thread
  {
    /* Owned by thread.c. */
    tid_t tid;                          /* Thread identifier. */
    enum thread_status status;          /* Thread state. */
    char name[16];                      /* Name (for debugging purposes). */
    uint8_t *stack;                     /* Saved stack pointer. */
    int priority;                       /* Priority. */
    struct list_elem allelem;           /* List element for all threads list. */

    /* Shared between thread.c and synch.c. */
    struct list_elem elem;              /* List element. */

#ifdef USERPROG
    /* Owned by userprog/process.c. */
    uint32_t *pagedir;                  /* Page directory. */
#endif

    /* Owned by thread.c. */
    unsigned magic;                     /* Detects stack overflow. */

    int64_t ticks_blocked;              /* Time for blocked. */
  };
```
添加了一个变量ticks_blocked用于记录剩余阻塞时间。在timer_sleep函数中，将该线程阻塞并设置阻塞时间。这一过程需要解除中断。  
修改timer_sleep函数
```
void
timer_sleep (int64_t ticks)
{
  if(ticks <= 0) return;
  ASSERT (intr_get_level () == INTR_ON);
  enum intr_level old_level = intr_disable ();
  struct thread *current_thread = thread_current ();
  current_thread->ticks_blocked = ticks;
  thread_block ();
  intr_set_level (old_level);
}
```
thread_block()的底层是将当前线程设置为THREAD_BLOCKED后重新调度，状态为THREAD_BLOCKED的线程将从就绪队列中移除。  
然后在适当时间唤醒线程，在每个tick内遍历所有线程，并将ticks_blocked值减一，如果该值小于等于0，则将其从阻塞队列中移除并重新调度。每次时间片轮转都会调度timer_interrupt函数。  
```
/* Timer interrupt handler. */
static void
timer_interrupt (struct intr_frame *args UNUSED)
{
  ticks++;
  thread_tick ();
  thread_foreach(check_blocked_time,NULL);
}
```
定义check_blocked_time函数：将ticks_blocked减一，若ticks_blocked小于0，则将其唤醒。
```void check_blocked_time(struct thread *t, void *aux){
  if (t->status == THREAD_BLOCKED && t->ticks_blocked > 0){
    t->ticks_blocked--;
    if (t->ticks_blocked == 0)
      thread_unblock(t);
  }
}

```
在头文件也要添加关于check_blocked_time的声明  
```
void check_blocked_time(struct thread *t, void *aux);
```  
此时Mission1通过部分  
```
pass tests/threads/alarm-single
pass tests/threads/alarm-multiple
pass tests/threads/alarm-simultaneous
FAIL tests/threads/alarm-priority
pass tests/threads/alarm-zero
pass tests/threads/alarm-negative
```
然后完成线程优先级的问题。通过发现thread.c中的next_thread_to_run()函数，发现其中存在一个ready_list。  
```
static struct thread *
next_thread_to_run (void)
{
  if (list_empty (&ready_list))
    return idle_thread;
  else
    return list_entry (list_pop_front (&ready_list), struct thread, elem);
}
```
查找list.c文件，发现了list_max函数，可用于根据比较函数查找ready_list中优先级最高的线程。构造比较函数，利用list_max和list_entry将优先级最高的线程移除并返回。
```
bool thread_compare_priority (const struct list_elem *a,const struct list_elem *b,void *aux UNUSED){
  return list_entry(a,struct thread,elem)->priority < list_entry(b,struct thread,elem)->priority;
}
static struct thread *
next_thread_to_run (void)
{
  if (list_empty (&ready_list))
    return idle_thread;
  else{
    struct list_elem *max_priority = list_max (&ready_list,thread_compare_priority,NULL);
    list_remove (max_priority);
    return list_entry (max_priority,struct thread,elem);
  }
}
```
此时，Mission1部分全部通过。  
```
pass tests/threads/alarm-single
pass tests/threads/alarm-multiple
pass tests/threads/alarm-simultaneous
pass tests/threads/alarm-priority
pass tests/threads/alarm-zero
pass tests/threads/alarm-negative
```
#### Mission2
从priority-fifo测试看起，改测试创建了一个一个优先级PRI_DEFAULT+2的主线程，并用这个线程创建了16个优先级PRI_DEFAULT+1的子线程，然后把主线程的优先级设置为优先级PRI_DEFAULT。  
测试需要把16个线程跑完后结束主线程，但操作系统中线程是并行执行的，有可能最开始的一个线程在设置完优先级之后立刻结束了，而此时其他线程并未结束。因此在线程设置完优先级之后应该立刻重新调度，需要在thread_set_priority()函数里添加thread_yield()函数。  
```
/* Sets the current thread's priority to NEW_PRIORITY. */
void
thread_set_priority (int new_priority)
{
  thread_current ()->priority = new_priority;
  thread_yield();
}
```
对于priority-preempt测试，要求创建一个新的高优先级线程抢占当前线程，在thread_create()中加一个判断：如果新线程的优先级高于当前线程优先级，调用thread_yield()函数。
```
tid_t
thread_create (const char *name, int priority,
               thread_func *function, void *aux)
{
  struct thread *t;
  struct kernel_thread_frame *kf;
  struct switch_entry_frame *ef;
  struct switch_threads_frame *sf;
  tid_t tid;

  ASSERT (function != NULL);

  /* Allocate thread. */
  t = palloc_get_page (PAL_ZERO);
  if (t == NULL)
    return TID_ERROR;

  /* Initialize thread. */
  init_thread (t, name, priority);
  tid = t->tid = allocate_tid ();

  /* Stack frame for kernel_thread(). */
  kf = alloc_frame (t, sizeof *kf);
  kf->eip = NULL;
  kf->function = function;
  kf->aux = aux;

  /* Stack frame for switch_entry(). */
  ef = alloc_frame (t, sizeof *ef);
  ef->eip = (void (*) (void)) kernel_thread;

  /* Stack frame for switch_threads(). */
  sf = alloc_frame (t, sizeof *sf);
  sf->eip = switch_entry;
  sf->ebp = 0;

  /* Add to run queue. */
  thread_unblock (t);

  if (thread_current ()->priority < priority)
    thread_yield ();

  return tid;
}
```
完成以上同时也完成了priority-change测试（该测试要求创建一个新线程并立刻调用，在降低优先级后不再继续执行）。  
处理线程同步问题：  
priority-seme测试：创建了10个优先级不等的线程，每个线程调用sema_down函数，其他得不到信号量的线程都得阻塞，而每次运行的线程释放信号量时必须确保优先级最高的线程继续执行。   
修改sema_up函数，在waiters中取出优先级最高的thread,调用thread_yield()。  
```
void
sema_up (struct semaphore *sema) 
{
  enum intr_level old_level;

  ASSERT (sema != NULL);

  old_level = intr_disable ();
  if (!list_empty (&sema->waiters)) {
    struct list_elem *max_priority = list_max (&sema->waiters,thread_compare_priority,NULL);
    list_remove (max_priority);
    thread_unblock(list_entry (max_priority,struct thread,elem));
  }18   sema->value++;
  intr_set_level (old_level);
  thread_yield();
}
```
priority-condvar测试：条件变量维护了一个waiters用于存储等待接受条件变量的线程，所以我们只需修改cond_signal()函数唤醒最高优先级的线程即可。这里需要我们自己新建一个比较函数cond_compare_priority。  
```
bool cond_compare_priority (const struct list_elem *a,const struct list_elem *b,void *aux UNUSED){
  struct semaphore_elem *sa = list_entry(a,struct semaphore_elem,elem);
  struct semaphore_elem *sb = list_entry(b,struct semaphore_elem,elem);
  return list_entry(list_front(&sa->semaphore.waiters),struct thread,elem)->priority <
         list_entry(list_front(&sb->semaphore.waiters),struct thread,elem)->priority;
}

/* If any threads are waiting on COND (protected by LOCK), then
   this function signals one of them to wake up from its wait.
   LOCK must be held before calling this function.

   An interrupt handler cannot acquire a lock, so it does not
   make sense to try to signal a condition variable within an
   interrupt handler. */
void
cond_signal (struct condition *cond, struct lock *lock UNUSED)
{
  ASSERT (cond != NULL);
  ASSERT (lock != NULL);
  ASSERT (!intr_context ());
  ASSERT (lock_held_by_current_thread (lock));

  if (!list_empty (&cond->waiters)){
    struct list_elem *max_priority = list_max (&cond->waiters,cond_compare_priority,NULL);
    list_remove (max_priority);
    sema_up(&list_entry(max_priority,struct semaphore_elem,elem)->semaphore);
  }
}
```
处理优先级捐赠问题：  
关于优先级捐赠共七个测试，其中priority-donate-multiple测试提出一个线程可能有多个锁，其余线程会因为该线程而阻塞；priority-donate-chain测试提出优先级嵌套的问题，优先级的更新操作是需要循环的，而循环的关键点是知道当前锁的拥有者。  
因此，首先修改thread和lock结构体：   

```
struct thread
  {
    /* Owned by thread.c. */
    tid_t tid;                          /* Thread identifier. */
    enum thread_status status;          /* Thread state. */
    char name[16];                      /* Name (for debugging purposes). */
    uint8_t *stack;                     /* Saved stack pointer. */
    int priority;                       /* Priority. */
    struct list_elem allelem;           /* List element for all threads list. */

    /* Shared between thread.c and synch.c. */
    struct list_elem elem;              /* List element. */

#ifdef USERPROG
    /* Owned by userprog/process.c. */
    uint32_t *pagedir;                  /* Page directory. */
#endif

    /* Owned by thread.c. */
    unsigned magic;                     /* Detects stack overflow. */

    int64_t ticks_blocked;              /* Time for blocked. */
    struct list locks;                  /* Locks this thread holds */
    struct lock *waiting_lock;          /* The lock this thread is waiting for */
    int original_priority;              /* Original priority of this thread */
  };
```
```
struct lock
  {
    int max_priority;           /* Max priority of all threads aquiring this lock */
    struct list_elem elem;      /* Used in thread.c */
    struct thread *holder;      /* Thread holding lock (for debugging). */
    struct semaphore semaphore; /* Binary semaphore controlling access. */
  };
```
修改lock_acquire()函数：在获取锁之前，循环更新所有参与嵌套的线程的优先级。  
```
void
lock_acquire (struct lock *lock)
{
  ASSERT (lock != NULL);
  ASSERT (!intr_context ());
  ASSERT (!lock_held_by_current_thread (lock));

  if(lock->holder != NULL && !thread_mlfqs){
    thread_current()->waiting_lock = lock;
    struct lock *wlock = lock;
    while(wlock != NULL && thread_current()->priority > wlock->max_priority){
      wlock->max_priority = thread_current()->priority;
      struct list_elem *max_priority_in_locks = list_max(&wlock->holder->locks,lock_compare_max_priority,NULL);
      int maximal = list_entry(max_priority_in_locks,struct lock,elem)->max_priority;
      if(wlock->holder->priority < maximal)
        wlock->holder->priority = maximal;
      wlock = wlock->holder->waiting_lock;
    }
  }

  sema_down (&lock->semaphore);
  lock->holder = thread_current ();

  if(!thread_mlfqs){
    thread_current()->waiting_lock = NULL;
    lock->max_priority = thread_current()->priority;
    list_push_back(&thread_current()->locks,&lock->elem);
    if(lock->max_priority > thread_current()->priority){
      thread_current()->priority = lock->max_priority;
      thread_yield();
    }
  }
}
```
修改lock_release()函数：在释放锁之前，对该线程的优先级进行更新，若该线程没有拥有的锁，就直接更新为original_priority，否则从所有锁的max_priority中找到最大值进行更新。   
```
void
lock_release (struct lock *lock)
{
  ASSERT (lock != NULL);
  ASSERT (lock_held_by_current_thread (lock));

  if(!thread_mlfqs){
    list_remove(&lock->elem);
    int maximal = thread_current()->original_priority;
    if(!list_empty(&thread_current()->locks)){
      struct list_elem *max_priority_in_locks = list_max(&thread_current()->locks,lock_compare_max_priority,NULL);
      int p = list_entry(max_priority_in_locks,struct lock,elem)->max_priority;
      if(p > maximal)
        maximal = p;
    }
    thread_current()->priority = maximal;
  }

  lock->holder = NULL;
  sema_up (&lock->semaphore);
}
```
修改thread_set_priority(int new_priority)函 数：若没有锁则直接更新不考虑优先级捐赠情况，或当更新优先级大于当前线程优先级，更新。同时，original_priority总需要更新。  
同时也需要修改list_max中的比较函数。
```
void
thread_set_priority (int new_priority)
{
  thread_current ()->original_priority = new_priority;
  if(list_empty(&thread_current()->locks) || new_priority > thread_current()->priority){
    thread_current()->priority = new_priority;
    thread_yield();
  }
}
```
```
bool lock_compare_max_priority (const struct list_elem *a,const struct list_elem *b,void *aux UNUSED){
  return list_entry(a,struct lock,elem)->max_priority < list_entry(b,struct lock,elem)->max_priority;
}
```
此时，Mission2部分全部通过：  
```
pass tests/threads/priority-change
pass tests/threads/priority-donate-one
pass tests/threads/priority-donate-multiple
pass tests/threads/priority-donate-multiple2
pass tests/threads/priority-donate-nest
pass tests/threads/priority-donate-sema
pass tests/threads/priority-donate-lower
pass tests/threads/priority-fifo
pass tests/threads/priority-preempt
pass tests/threads/priority-sema
pass tests/threads/priority-condvar
pass tests/threads/priority-donate-chain
```
#### Mission3
该部分测试主要实现多级反馈队列调度算法，文档中提到：  
> Unfortunately, Pintos does not support floating-point arithmeticin the kernel, because it would complicate and slow the kernel. 

我需要先部署该浮点类的计算部分。由文档可知，编写fixed-point.h文件。
```
#ifndef FIXED_POINT_H
#define FIXED_POINT_H

#define p 17
#define q 14
#define f (1<<q)

#define CONVERT_N_TO_FIXED_POINT(n)             ((n)*(f))
#define CONVERT_X_TO_INTEGER_ZERO(x)            ((x)/(f))
#define CONVERT_X_TO_INTEGER_NEAREST(x)         (((x)>=0)?(((x)+(f)/2)/(f)):(((x)-(f)/2)/(f)))

#define ADD_X_AND_Y(x,y)                        ((x)+(y))
#define SUBTRACT_Y_FROM_X(x,y)                  ((x)-(y))
#define ADD_X_AND_N(x,n)                        ((x)+(n)*(f))
#define SUBTRACT_N_FROM_X(x,n)                  ((x)-(n)*(f))
#define MULTIPLY_X_BY_Y(x,y)                    (((int64_t) (x))*(y)/(f))
#define MULTIPLY_X_BY_N(x,n)                    ((x)*(n))
#define DIVIDE_X_BY_Y(x,y)                      (((int64_t) (x))*(f)/(y))
#define DIVIDE_X_BY_N(x,n)                      ((x)/(n))

#endif
```
然后需要实现一个算法。通过文档可以大致了解该算法内容：
> 1. 该算法的优先级是动态变化的，主要动态修改Niceness, Priority, recent_cpu, load_avg四大变量
> 
> 2. Priority的计算公式为：priority= PRI_MAX - (recent_cpu/ 4) - (nice*2)，每四个clock tick对所有线程更新一次
>
> 3. recent_cpu的计算公式为recent_cpu= (2*load_avg)/(2*load_avg+ 1) *recent_cpu+nice，当timer_ticks () % TIMER_FREQ == 0时对所有线程更新，每个tick对当前线程的recent_cpu加1。
> 
> 4. load_avg的计算公式为load_avg= (59/60)*load_avg+ (1/60)*ready_threads，当timer_ticks () % TIMER_FREQ == 0时对所有线程更新

首先，修改thread结构体，添加成员变量nice、recent_cpu：
```
struct thread
  {
    /* Owned by thread.c. */
    tid_t tid;                          /* Thread identifier. */
    enum thread_status status;          /* Thread state. */
    char name[16];                      /* Name (for debugging purposes). */
    uint8_t *stack;                     /* Saved stack pointer. */
    int priority;                       /* Priority. */
    struct list_elem allelem;           /* List element for all threads list. */

    /* Shared between thread.c and synch.c. */
    struct list_elem elem;              /* List element. */

#ifdef USERPROG
    /* Owned by userprog/process.c. */
    uint32_t *pagedir;                  /* Page directory. */
#endif

    /* Owned by thread.c. */
    unsigned magic;                     /* Detects stack overflow. */

    int64_t ticks_blocked;              /* Time for blocked. */
    struct list locks;                  /* Locks this thread holds */
    struct lock *waiting_lock;          /* The lock this thread is waiting for */
    int original_priority;              /* Original priority of this thread */

    int nice;                           /* Niceness of thread used in mlfqs */
    int64_t recent_cpu;                 /* Used in mlfqs */
  };
```
在thread.c中定义全局变量load_avg，同时编写下列函数并在头文件声明：
```
int64_t load_avg;
```
```
/* Increment by 1 for each clock tick */
void increase_recent_cpu(void){
  if (thread_current()!=idle_thread)
    thread_current()->recent_cpu = ADD_X_AND_N(thread_current()->recent_cpu,1);
}

/* Modify Priority */
void modify_priority(struct thread *t,void *aux UNUSED){
  if (t!=idle_thread){
    //priority = PRI_MAX - (recent_cpu / 4) - (nice * 2)
    t->priority = CONVERT_X_TO_INTEGER_NEAREST(CONVERT_N_TO_FIXED_POINT(PRI_MAX)-
    t->recent_cpu/4-CONVERT_N_TO_FIXED_POINT(2*t->nice));
    if (t->priority < PRI_MIN)
      t->priority = PRI_MIN;
    if (t->priority > PRI_MAX)
      t->priority = PRI_MAX;
  }
}

/* Modify recent_cpu */
void modify_cpu(struct thread *t,void *aux UNUSED){
  if (t != idle_thread){
  int64_t fa = MULTIPLY_X_BY_N(load_avg,2);
  int64_t fb = MULTIPLY_X_BY_N(load_avg,2)+CONVERT_N_TO_FIXED_POINT(1);
  t->recent_cpu = MULTIPLY_X_BY_Y(DIVIDE_X_BY_Y(fa,fb),t->recent_cpu)+
  CONVERT_N_TO_FIXED_POINT(t->nice);
  }
}

/* Modify load average */
void modify_load_avg(void){
  int ready_threads = list_size(&ready_list);
  if (thread_current()!=idle_thread){
    ready_threads++;
  }
  int64_t fa = MULTIPLY_X_BY_N(load_avg,59);
  int add1 = DIVIDE_X_BY_N(fa,60);
  int add2 = DIVIDE_X_BY_N(CONVERT_N_TO_FIXED_POINT(ready_threads),60);
  load_avg = ADD_X_AND_Y(add1,add2);
}
```
修改timer.c文件，保证每次中断时都对这些值进行更新。
```
/* Timer interrupt handler. */
static void
timer_interrupt (struct intr_frame *args UNUSED)
{
  ticks++;
  thread_tick ();
  thread_foreach(check_blocked_time,NULL);

  if(thread_mlfqs){
    increase_recent_cpu();
    if (timer_ticks() % TIMER_FREQ == 0){
      modify_load_avg();
      thread_foreach(modify_cpu,NULL);
    }
    if (timer_ticks() % 4 == 0){
      thread_foreach(modify_priority,NULL);
    }
  }
}
```
最后把原框架函数补全：
```
/* Sets the current thread's nice value to NICE. */
void
thread_set_nice (int nice UNUSED)
{
  thread_current()->nice = nice;
  modify_priority(thread_current(),NULL);
  thread_yield();
}

/* Returns the current thread's nice value. */
int
thread_get_nice (void)
{
  return thread_current()->nice;
}

/* Returns 100 times the system load average. */
int
thread_get_load_avg (void)
{
  int temp = MULTIPLY_X_BY_N(load_avg,100);
  return CONVERT_X_TO_INTEGER_NEAREST(temp);
}

/* Returns 100 times the current thread's recent_cpu value. */
int
thread_get_recent_cpu (void)
{
  return CONVERT_X_TO_INTEGER_NEAREST(MULTIPLY_X_BY_N(thread_current()->recent_cpu,100));
}
```
至此，project1所有测试通过！
```
pass tests/threads/alarm-single
pass tests/threads/alarm-multiple
pass tests/threads/alarm-simultaneous
pass tests/threads/alarm-priority
pass tests/threads/alarm-zero
pass tests/threads/alarm-negative
pass tests/threads/priority-change
pass tests/threads/priority-donate-one
pass tests/threads/priority-donate-multiple
pass tests/threads/priority-donate-multiple2
pass tests/threads/priority-donate-nest
pass tests/threads/priority-donate-sema
pass tests/threads/priority-donate-lower
pass tests/threads/priority-fifo
pass tests/threads/priority-preempt
pass tests/threads/priority-sema
pass tests/threads/priority-condvar
pass tests/threads/priority-donate-chain
pass tests/threads/mlfqs-load-1
pass tests/threads/mlfqs-load-60
pass tests/threads/mlfqs-load-avg
pass tests/threads/mlfqs-recent-1
pass tests/threads/mlfqs-fair-2
pass tests/threads/mlfqs-fair-20
pass tests/threads/mlfqs-nice-2
pass tests/threads/mlfqs-nice-10
pass tests/threads/mlfqs-block
All 27 tests passsed.
```
### 实验感想
这次操作系统课程设计的难度很大，无论是从环境的配置和搭建，文档的查询和翻译，相关资料的搜集和理解，都需要花费很多的功夫。也感谢老师一开学就提前的提醒让我们及早开始着手才不至于让我在最后期末几天里捉襟见肘。   
首先需要说明的是，在课程资料里提供的pintos安装教程是错误的，虽然可以安装成功，但是首次运行27个测试全部failed，这是不合理的，因为按照标准测试结果，即使是原封不动的代码也会有一部分测试是可以通过的。并且我根据教程去尝试修改thread第一部分内容后make check仍然是全部failed，没有丝毫改变。经过资料的搜寻，我发现是因为驱动程序qemu/bochs的选择原因。**教程中选用的模拟器是bochs但在教程中却没有给出bochs的安装提示，这导致无论如何都无法通过任何测试。**  
后来根据网上的完整版教程，最终我还是安装成功，在这里附上链接，希望可以对下一届同学有所帮助，能够少绕一些弯路。
> https://blog.csdn.net/weixin_43573233/article/details/111708642

总体来说这次课设的难度还是很高的，实话实说，能够完全理解thread的内容就已经耗费了我很长的时间。在课堂上学习的大多只是非常理论化的知识，**而在pintos中切实去完成一套线程调度的系统才真正让我理解了操作系统中的线程到底是怎么一回事，需要考虑哪些因素：优先级的循环更新，嵌套调度，时间片的考量，关于线程锁的概念，以及更复杂的队列调度算法。** 这让我很难不生起完成全部的pintos内容的想法，那样的话一定会对操作系统有着更为全面而深刻认知。但受限于时间原因，这次课设很可惜只能完成到thread部分。   
  
最后，感谢老师一学期的授课，预祝老师新年快乐！
