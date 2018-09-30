####pthread和GCD比较分析(一)

通过上一篇我们分析gcd的源码，我清楚的知道，gcd的内部实现核心逻辑基本上是通过pthread来实现的
这一篇我们主要来讲pthread的几个比较常用的方法和一些概念的介绍，然后再介绍pthread和gcd实现相同功能的不同写法

#####什么是线程？

在理解线程之前，应先对UNIX进程（process）有所了解。进程被操作系统创建，需要相当多的“额外开销”。进程包含了程序的资源和执行状态信息。如下： 
进程ID，进程group ID，用户ID和group ID 
环境 
工作目录  
程序指令 
寄存器 
栈 
堆 
文件描述符 
信号动作（Signal actions） 
共享库 
进程间通信工具（如：消息队列，管道，信号量或共享内存）

线程和进程十分相似，不同的只是线程比进程小。首先，线程采用了多个线程可共享资源的设计思想；例如，它们的操作大部分都是在同一地址空间进行的。其次，从一个线程切换到另一线程所花费的代价比进程低。再次，进程本身的信息在内存中占用的空间比线程大，因此线程更能允分地利用内存。

#####什么是Pthreads

Pthreads是实现多线程的私有库，可以提高潜在的系统的性能。

#####创建和结束线程： pthread_create

```
/*最初，main函数包含了一个缺省的线程。其它线程则需要程序员显式地创建。 
pthread_create 创建一个新线程并使之运行起来。该函数可以在程序的任何地方调用。 
pthread_create参数： 
thread：返回一个不透明的，唯一的新线程标识符。 
attr：不透明的线程属性对象。可以指定一个线程属性对象，或者NULL为缺省值。 
start_routine：线程将会执行一次的C函数。 
arg: 传递给start_routine单个参数，传递时必须转换成指向void的指针类型。没有参数传递时，可设置为NULL。 
一个进程可以创建的线程最大数量取决于系统实现。 
一旦创建，线程就称为peers，可以创建其它线程。线程之间没有指定的结构和依赖关系。*/

pthread_create (thread,attr,start_routine,arg)  

// 结束终止
pthread_exit (status)  


/*线程被创建时会带有默认的属性。其中的一些属性可以被程序员用线程属性对象来修改。 
pthread_attr_init 和 pthread_attr_destroy用于初始化/销毁先成属性对象。 
其它的一些函数用于查询和设置线程属性对象的指定属性。*/
pthread_attr_init (attr)  

pthread_attr_destroy (attr) 
```
下面我们来看一下具体的创建线程的实例
```
#include <stdio.h>
#include <pthread.h>
#include <stdio.h>

#define NUM_THREADS     5

void *PrintHello(void *threadid) {
    int tid;
    tid = (int)threadid;
    printf("Hello World! It's me, thread #%d!\n", tid);
    pthread_exit(NULL);// 创建的线程进行释放
}

int main (int argc, char *argv[]) {
    
    pthread_t threads[NUM_THREADS];
    
    int rc, t;
    
    for(t=0; t<NUM_THREADS; t++){
        
        printf("In main: creating thread %d\n", t);
        
        rc = pthread_create(&threads[t], NULL, PrintHello, (void *)t);
        
        if (rc){
            printf("ERROR; return code from pthread_create() is %d\n", rc);
        }
        
    }
    
    pthread_exit(NULL);// 创建的线程一定的记得进行释放，一是线程的创建也会占用一部分的资源，二是，你永远不知道这些创建的线程一直存在，会造成什么样的影响
}
```

#####向线程传递参数
- pthread_create()函数允许程序员想线程的start routine传递一个参数。当多个参数需要被传递时，可以通过定义一个结构体包含所有要传的参数，然后用pthread_create()传递一个指向改结构体的指针，来打破传递参数的个数的限制。 

- 所有参数都应该传引用传递并转化成（void*）。

我们直接看例子：

```objectivec

/*
下面的代码片段演示了如何向一个线程传递一个简单的整数。主线程为每一个线程使用一个唯一的数据结构，确保每个线程传递的参数是完整的。
*/

int *taskids[NUM_THREADS]; 
for(t=0; t<NUM_THREADS; t++) { 
   taskids[t] = (int *) malloc(sizeof(int)); 

   *taskids[t] = t; 

   printf("Creating thread %d\n", t); 

   rc = pthread_create(&threads[t], NULL, PrintHello,  

        (void *) taskids[t]); 

   ... 

} 
```

```
/*
例子展示了用结构体向线程设置/传递参数。每个线程获得一个唯一的结构体实例。 
*/
struct thread_data{ 
   int  thread_id; 
   int  sum; 
   char *message; 
}; 

struct thread_data thread_data_array[NUM_THREADS]; 

void *PrintHello(void *threadarg) 
{ 
   struct thread_data *my_data; 

   ... 

   my_data = (struct thread_data *) threadarg; 
   taskid = my_data->thread_id; 
   sum = my_data->sum; 
   hello_msg = my_data->message; 
   ... 

} 

int main (int argc, char *argv[]) 
{ 

   ... 

   thread_data_array[t].thread_id = t; 

   thread_data_array[t].sum = sum; 

   thread_data_array[t].message = messages[t]; 

   rc = pthread_create(&threads[t], NULL, PrintHello,  

        (void *) &thread_data_array[t]); 

   ... 

} 
```

```
/*
	例子演示了错误地传递参数。循环会在线程访问传递的参数前改变传递给线程的地址的内容。 
*/

int rc, t; 

for(t=0; t<NUM_THREADS; t++)  { 
   printf("Creating thread %d\n", t); 
   rc = pthread_create(&threads[t], NULL, PrintHello,  (void *) &t); 

} 
```
#####连接（Joining）和分离（Detaching）线程 
```

pthread_join (threadid,status)  // 线程同步使用的，等待线程执行完，再继续执行其他

pthread_detach (threadid,status)  

pthread_attr_setdetachstate (attr,detachstate)  

pthread_attr_getdetachstate (attr,detachstate) 
```
先看例子，在后面的其他例子中也有体现
```
void * thread_function(void *arg) {
    int * incoming = (int *)arg;
    printf("this is in pthread and arg is %d\n", *incoming);
    return NULL;
}
int main (int argc, char *argv[]) {
	pthread_t thread_id ;
    int value = 63;
    pthread_create(&thread_id, NULL, thread_function, &value);
    pthread_join(thread_id, NULL);
    NSLog(@"11111");
    pthread_exit(NULL);
}
```
pthread_join“连接”是一种在线程间完成同步的方法，以上面的例子，这是一个同步执行的操作，先执行thread_function，在执行后面的NSLog

TODO: Detaching

#####互斥量（Mutex Variables）
互斥量（Mutex）是“mutual exclusion”的缩写。互斥量是实现线程同步，和保护同时写共享数据的主要方法 .
是避免线程间相互交叠的一种方法。可以把它想像成一个惟一的物体，必须把它收藏好，但是只有别人都不占有它时您才可以占有它，在您主动放弃它之前也没有人可以占有它。占有这个惟一物体的过程就叫做锁定或者获得互斥量。不同的人学到的对这这件事的叫法不同，所以当您和别人谈到这件事时别人可能会用不同的词来表达。POSIX 线程接口沿用相当一致的术语，所以称为“锁定”。
互斥量对共享数据的保护就像一把锁。在Pthreads中，任何时候仅有一个线程可以锁定互斥量，因此，当多个线程尝试去锁定该互斥量时仅有一个会成功。直到锁定互斥量的线程解锁互斥量后，其他线程才可以去锁定互斥量。线程必须轮着访问受保护数据。

```
pthread_mutex_init (mutex,attr)  // 互斥量的初始化

pthread_mutex_destroy (mutex)  //互斥量销毁

pthread_mutexattr_init (attr)  // 创建互斥量属性对象的初始化

pthread_mutexattr_destroy (attr)  //创建互斥量属性对象的销毁
```
互斥量必须用类型pthread_mutex_t类型声明，在使用前必须初始化，这里有两种方法可以初始化互斥量：

- 声明时静态地，如：
pthread_mutex_t mymutex = PTHREAD_MUTEX_INITIALIZER;  

- 动态地用pthread_mutex_init()函数，这种方法允许设定互斥量的属性对象attr。 

```

pthread_mutex_lock (mutex)  // 互斥锁

pthread_mutex_trylock (mutex)  

pthread_mutex_unlock (mutex) //互斥解锁
```
我们先接着将下面的条件变量，因为这两种一般也会同时使用，在后面的例子中，我们也会一起解释

#####条件变量
条件变量提供了另一种同步的方式，它是通过wait和Signaling，也就是等待和发送信号的方式，来激活或者是等待已有的线程。我们来举一个场景：如果线程正在等待某个特定条件发生，它应该如何处理这种情况？它可以重复对互斥对象锁定和解锁，每次都会检查共享数据结构，以查找某个值。但这是在浪费时间和资源，而且这种繁忙查询的效率非常低。解决这个问题的最佳方法是使用 pthread_cond_wait() 调用来等待特殊条件发生。

pthread_cond_wait() 所做的第一件事就是同时对互斥对象解锁（于是其它线程可以修改已链接列表），并等待条件 mycond 发生（这样当 pthread_cond_wait() 接收到另一个线程的“信号”时，它将苏醒）。现在互斥对象已被解锁，其它线程可以访问和修改已链接列表，可能还会添加项。

此时，pthread_cond_wait() 调用还未返回。对互斥对象解锁会立即发生，但等待条件 mycond 通常是一个阻塞操作，这意味着线程将睡眠，在它苏醒之前不会消耗 CPU 周期。这正是我们期待发生的情况。线程将一直睡眠，直到特定条件发生，在这期间不会发生任何浪费 CPU 时间的繁忙查询。从线程的角度来看，它只是在等待 pthread_cond_wait() 调用返回。

现在继续说明，假设另一个线程（称作“2 号线程”）锁定了 mymutex 并对已链接列表添加了一项。在对互斥对象解锁之后，2 号线程会立即调用函数 pthread_cond_broadcast(&mycond)。此操作之后，2 号线程将使所有等待 mycond 条件变量的线程立即苏醒。这意味着第一个线程（仍处于 pthread_cond_wait() 调用中）现在将苏醒。

现在，看一下第一个线程发生了什么。您可能会认为在 2 号线程调用 pthread_cond_broadcast(&mymutex) 之后，1 号线程的 pthread_cond_wait() 会立即返回。不是那样！实际上，pthread_cond_wait() 将执行最后一个操作：重新锁定 mymutex。一旦 pthread_cond_wait() 锁定了互斥对象，那么它将返回并允许 1 号线程继续执行。那时，它可以马上检查列表，查看它所感兴趣的更改。

我们来回顾一下刚才的流程

第一个线程首先调用：
```
pthread_mutex_lock(&mymutex);
```

然后，它检查了列表。没有找到感兴趣的东西，于是它调用：
```
pthread_cond_wait(&mycond, &mymutex);
```
然后，pthread_cond_wait() 调用在返回前执行许多操作：
```
pthread_mutex_unlock(&mymutex);
```

#####创建和销毁条件变量 
```
pthread_cond_init (condition,attr)  

pthread_cond_destroy (condition)  

pthread_condattr_init (attr)  

pthread_condattr_destroy (attr)  
```
条件变量必须声明为pthread_cond_t类型，必须在使用前初始化。有两种方式可以初始条件变量： 
- 声明时静态地。如：
pthread_cond_t myconvar = PTHREAD_COND_INITIALIZER;  
- 用pthread_cond_init()函数动态地。创建的条件变量ID通过condition参数返回给调用线程。该方式允许设置条件变量对象的属性，attr。 

#####在条件变量上等待（Waiting）和发送信号（Signaling）
```
pthread_cond_wait (condition,mutex)  //pthread_cond_wait()阻塞调用线程直到指定的条件受信（signaled）。该函数应该在互斥量锁定时调用，当在等待时会自动解锁互斥量。在信号被发送，线程被激活后，互斥量会自动被锁定，当线程结束时，由程序员负责解锁互斥量。

pthread_cond_signal (condition)  //在某些情况下，活动线程只需要唤醒第一个正在睡眠的线程。假设您只对队列添加了一个工作作业。那么只需要唤醒一个工作程序线程（再唤醒其它线程是不礼貌的！）

pthread_cond_broadcast (condition)  //对于发送信号和广播，需要注意一点。如果线程更改某些共享数据，而且它想要唤醒所有正在等待的线程，则应使用 pthread_cond_broadcast 调用
```

下面，我们来看一下，一个整体的例子
```cpp
#include "pthreadWait.h"
#include <pthread.h>
#include <stdio.h>

#define NUM_THREADS  3
#define TCOUNT 10
#define COUNT_LIMIT 12

int count = 0;

int thread_ids[3] = {0,1,2};

pthread_mutex_t count_mutex; // 互斥量必须用类型pthread_mutex_t类型声明, 必须在使用前进行初始化

pthread_cond_t count_threshold_cv; //条件变量必须声明为pthread_cond_t类型，必须在使用前进行初始化

// 创建线程的时候，会执行的方法。在创建线程的时候，pthread_create一个参数，线程将会执行一次的C函数。
void *inc_count(void *idp) {
    
    int j,i;
    
    double result=0.0;
    
    int *my_id = idp;
    
    for (i=0; i<TCOUNT; i++) {
        // 互斥量，其实也是一种锁
        pthread_mutex_lock(&count_mutex);
        
        count++;
        
        /*
         Check the value of count and signal waiting thread when condition is
         reached.  Note that this occurs while mutex is locked.
         */
        
        if (count == COUNT_LIMIT) {
            // 当打到count == COUNT_LIMIT的时候，像count_threshold_cv发送信号
            pthread_cond_signal(&count_threshold_cv);
            printf("inc_count(): thread %d, count = %d  Threshold reached.\n",
                   *my_id, count);
            
        }
        
        printf("inc_count(): thread %d, count = %d, unlocking mutex\n",*my_id, count);
        
        // 一个流程结束，要进行解锁
        pthread_mutex_unlock(&count_mutex);
        
        /* Do some work so threads can alternate on mutex lock */
        for (j=0; j<1000; j++) {
            result = result + j;
        }
        
    }
    // 要把当前线程进行释放
    pthread_exit(NULL);
    
}

void *watch_count(void *idp) {
    int *my_id = idp;
    
    printf("Starting watch_count(): thread %d\n", *my_id);
    
    /*
     
     Lock mutex and wait for signal.  Note that the pthread_cond_wait
     
     routine will automatically and atomically unlock mutex while it waits.
     
     Also, note that if COUNT_LIMIT is reached before this routine is run by
     
     the waiting thread, the loop will be skipped to prevent pthread_cond_wait
     
     from never returning.
     
     */
    // 在pthread_cond_wait之前，一定首先对互斥量的锁
    pthread_mutex_lock(&count_mutex);
    
    if (count<COUNT_LIMIT) {
        // 在这里分析，当count<COUNT_LIMIT的时候，其中一个线程进入到这里，当前线程会进入等待状态，这个时候，此线程是休眠状态，从性能的角度分析，是没有占用过多cpu，当对此count_threshold_cv条件变量执行发送信号之后，也就是执行pthread_cond_signal(&count_threshold_cv)之后，会再次执行此方法，进行加锁和解锁
        pthread_cond_wait(&count_threshold_cv, &count_mutex);
        
        printf("watch_count(): thread %d Condition signalreceived.\n", *my_id);
        
    }
    // 结束之后，进行解锁
    pthread_mutex_unlock(&count_mutex);
    
    pthread_exit(NULL);
    
}

int main (int argc, char *argv[]) {
    
    int i, rc;
    
    pthread_t threads[3];
    
    pthread_attr_t attr; // 创建线程的属性，必须是此类型pthread_attr_t
    
    /* Initialize mutex and condition variable objects */
    
    pthread_mutex_init(&count_mutex, NULL); // 互斥量的初始化
    
    pthread_cond_init (&count_threshold_cv, NULL); // 条件量大初始化
    
    /* For portability, explicitly create threads in a joinable state */
    
    pthread_attr_init(&attr); //线程属性对象的初始化
    
    pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_JOINABLE); // 设置线程属性，设置线程为join，可连接的
    
    pthread_create(&threads[0], &attr, inc_count, (void *)&thread_ids[0]); // 分别创建线程
    
    pthread_create(&threads[1], &attr, inc_count, (void *)&thread_ids[1]); // 分别创建线程
    
    pthread_create(&threads[2], &attr, watch_count, (void *)&thread_ids[2]); // 分别创建线程
    
    /* Wait for all threads to complete */
    // 这个目的j就是等待所有的线程都执行完，pthread_join指向哪一个线程，也就是第一个参数，就会等待该线程执行完毕，才会继续执行，现在是for循环，把所有的线程都执行了join
    for (i=0; i<NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }
    
    printf ("Main(): Waited on %d  threads. Done.\n", NUM_THREADS);
    
    /* Clean up and exit */
    
    // 后面的操作都是需要改销毁的销毁，该g推出的推出。因为不退出，占用想用的资源不好，更重要的时候，你不知道你的 系统会出现什么问题
    pthread_attr_destroy(&attr);
    
    pthread_mutex_destroy(&count_mutex);
    
    pthread_cond_destroy(&count_threshold_cv);
    
    pthread_exit(NULL);
    
}
```
在上面的例子中，有很详细的注释

#####Reader/Writer Locks 读写锁
对于读写锁来说, 多个线程可以同时获得读锁, 但某一个时间内, 只有一个线程可以获得写锁. 如果已经有线程获得了读锁, 则任何请求写锁的线程将被阻塞在写锁函数的调用上, 同时如果线程已经获得了写锁, 那么任何请求读锁或者写锁 的线程都会被阻塞. 下面是读写锁的基本函数:
锁类型
```
pthread_rwlock_t
```
初始化/销毁
```
int pthread_rwlock_init(pthread_rwlock_t *restrict rwlock, const pthread_rwlockattr_t *restrict attr);
int pthread_rwlock_destroy(pthread_rwlock_t *rwlock);
```
读锁
```
int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock);
```
写锁
```
int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock);
```
释放锁
```
int pthread_rwlock_unlock(pthread_rwlock_t *rwlock);
```
例子：
```
pthread_rwlock_t rw_lock;
void * p_rwlock(void * arg) {
    pthread_rwlock_rdlock(&rw_lock);
    // 读取共享变量
    pthread_rwlock_unlock(&rw_lock);
    
} 
void  test_rwlock() {
    pthread_rwlock_init(&rw_lock, NULL);
    // 创建线程
    pthread_rwlock_wrlock(&rw_lock);
    // 修改共享变量
    pthread_rwlock_unlock(&rw_lock);
    // join线程
    pthread_rwlock_destroy(&rw_lock);
}
```
