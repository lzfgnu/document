# Ostep_cp30 条件变量

许多时候, 我们希望根据某个条件是否为真来决定线程是否继续运行. 比如说让 parent thread 在 child thread 运行结束后才继续运行(join 操作)



### 条件变量

* 定义

  条件变量是一个直观的队列. 当线程执行过程中的某些状态**(条件)**没有达到预期值的时候, 线程会被放入这个队列**(在这个队列上等待条件达到预期值)**. 对于其他的一些线程, 当它们改变条件值的时候, 可以唤醒在队列中等待的一个或多个线程, 从而使被唤醒的线程可以恢复执行.

  ***A condition variable is an explicit queue that threads can put themselves on when some state of execution (i.e., some condition) is not as desired (by waiting on the condition); some other thread, when it changes said state, can then wake one (or more) of those waiting threads and thus allow them to continue (by signaling on the condition)***

* 用例 --- 父线程等待子线程

  ```c
  int done = 0;
  pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
  pthread_cond_t c = PTHREAD_COND_INITIALIZER;
  
  void thread_exit() {
      pthread_mutex_lock(&m);
      done = 1;
      pthread_cond_signal(&c);
      pthread_mutex_unlock(&m);
  }
  
  void* child_rountine(void* arg) {
      printf("child\n");
      thread_exit();
      return NULL;
  }
  
  void thread_join() {
      pthread_mutex_lock(&m);
      while (done == 0) 
          pthread_cond_wait(&c, &m);
      pthread_mutex_unlock(&m);
  }
  
  int main() {
      printf("parent begin\n");
      pthread_t p;
      pthread_create(&p, NULL, child_routine, NULL);
      thread_join();
      printf("parent end\n");
      return 0;
  }
  ```

  #### 为什么调用pthread_cond_wait的时候要传递一把锁? 

  为了释放掉这把锁从而让其他的线程可以获取这把锁. 函数内部大概长这样:

  ```c
  pthread_cond_wait(pthread_mutex_t* m, pthred_cond_t* c) {
      pthread_mutex_unlock(m);
      // ...
      pthread_mutex_lock(m) // 离开函数时重新获取这把锁
  }
  ```

  



### 生产者/消费者问题

1. 错误的实现方式

   ```c
   int buf;
   int count = 0;
   
   void put(int val) {
       assert(count == 0);
       count = 1;
       buf = val;
   }
   
   int get() {
       assert(count == 1);
       count = 0;
       return buf;
   }
   
   cond_t cond;
   mutex_t mutex;
   
   void* producer(void* arg) {
       for (int i = 0; i < loops; ++i) {
           pthread_mutex_lock(&mutex);
           if (count == 1)
               pthread_cond_wait(&cond, &mutex);
           put(i);
           pthread_cond_signal(&cond);
           pthread_mutex_unlock(&mutex);
       }
   }
   
   void* consumer(void* arg) {
       for (int i = 0; i < loops; ++i) {
           pthread_mutex_lock(&mutex);
           if (count == 0) 
                                                               // 1
               pthread_cond_wait(&cond, &mutex);
           int tmp = get();
           pthread_cond_signal(&cond);
           pthread_mutex_unlock(&mutex);
           printf("%d\n", tmp);                                // 2
       }
   }
   ```

   该实现存在两个致命问题

   1. **在判断条件时使用了if   (为什么我们总是应该使用while来判断条件?)**

      假设有一个producer (p1) 和两个consumer (c1, c2).  c1最先执行, 执行到1的时候, 它进入sleep状态. 接着 p1 将buffer填充并唤醒c1 (**这之后count=1**), 与此同时c2立马执行到2的位置. 

      而此时c1也继续往下执行. **boom!!!** buffer 内只有一个值, 而这个值已经被c2读掉了, 而c1又要去读取这个值.

      如果将if改为while, 那么c1被唤醒后还会去检测条件值, 从而避免了上述情况.

   2. **只使用了一个条件变量**

      假设还是一个producer和两个consumer. c1, c2先运行, 执行到1的时候, 它们进入sleep状态. 这之后p1运行, 唤醒c1之后进入sleep状态. 此时线程状态: p1(sleep), c2(sleep), c1(running). 问题来了,  c1可能唤醒p1也可能唤醒c2. 如果唤醒c2, 那么整个程序将永远睡眠下去. 

      解决方案是使用两个条件变量或者一次唤醒所有的线程(cond_broadcast)

      

2. 正确的实现

   ```c
   cond_t empty, fill;
   mutex_t mutex;
   
   void* producer(void* arg) {
       for (int i = 0; i < loops; ++i) {
           pthread_mutex_lock(&mutex);
           while (count == 1) 
               pthread_cond_wait(&empty, &mutex):
           put(i);
           pthread_cond_signal(&fill);
           pthread_mutex_unlock(&mutex);
       }
   }
   void* consumer(void* arg) {
       for (int i = 0; i < loops; ++i) {
           pthread_mutex_lock(&mutex);
           while (count == 0) 
               pthread_cond_wait(&fill, &mutex);
           int tmp = get();
           pthread_cond_signal(&empty);
           pthread_mutex_unlock(&mutex);
           printf("%d\n", tmp);
       }
   }
   ```

3. 更泛用的版本

   ```c
   int buf[MAX];
   int fill_ptr = 0;
   int use_ptr = 0;
   iint count = 0;
   
   vooid put(int val) {
       buf[fill_ptr] = val;
       fill_ptr = (fill_ptr+1) % MAX;
       ++count;
   }
   
   void get() {
       int tmp = buffer[use_ptr];
       use_ptr = (use_ptr+1) % MAX;
       --count;
       return tmp;
   }
   
   cond_t empty, fill;
   mutex_t mutex;
   
   void * producer(void* arg) {
       for (int i = 0; i < loops; ++i) {
           pthread_mutex_lock(&mutex);
           while(count == MAX)
               pthread_cond_wait(&empty, &mutex);
           put(i);
           pthread_cond_signal(&fill);
          	pthread_mutex_unlock(&mutex);
       }
   }
   
   void* consumer(void* arg) {
       for (int i = 0; i < loops; ++i) {
           pthread_mutex_lock(&mutex);
           while (count == 0) 
               pthread_cond_wait(&fill, &mutex);
           int tmp = get();
           pthread_cond_signal(&empty);
           pthread_mutex_unlock(&mutex);
           printf("%d\n", tmp);
       }
   }
   ```



### Covering Conditions (cond_broadcast的应用场景)

有如下代码片段

```c
int bytes_left = MAX_HEAP_SIZE;

cond_t c;
mutex_t m;

void* allocate(int size) {
    pthread_mutex_lock(&m);
    while(bytes_left < size)
        pthread_cond_wait(&c, &m);
    void* ptr = ...;
    bytes_left -= size;
    pthread_mutex_unlock(&m);
    return ptr;
}

void free(void* ptr, int size) {
    pthread_mutex_lock(&m);
    bytes_left += size;
    pthread_cond_signal(&c);
    pthread_mutex_unlock(&m);
}
```

假设线程A allocate(100), 线程B allocate(10). A和B都因条件不足而进入sleep状态. 这时线程C free(50), 那么它会唤醒哪个线程呢? 很明显我们希望是B线程, 因为唤醒A线程将导致整个程序进入sleep状态. 可是C还就有可能会去唤醒A. 

对于这种问题的解决方案是使用pthread_cond_broadcast来唤醒所有sleep态的线程. 由于一下子唤醒了所有线程, 系统的性能可能会突然下降