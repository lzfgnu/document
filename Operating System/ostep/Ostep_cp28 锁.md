# Ostep_cp28 锁

### 锁的评估

锁的评估主要包含三个方面

* 互斥性. 这是最基本的, 防止多个线程同时进入临界区.
* 公正性. 尽可能保证每个线程都有机会获得锁.
* 性能. 单线程申请锁和多线程申请锁时的性能如何? 当cpu是多核时, 锁的性能又是如何?



### 控制中断(仅适用于单核cpu)

在单核cpu的时代, 临界区的互斥访问可以由开启/关闭中断来实现

```c
void lock() {
    DisableInterrupts();
}
void unlock() {
    EnableInterrupts();
}
```

在某个线程进入临界区后, os把终端关闭了, 确保了这个线程能不被中断的, 完整的走完这片临界区.

该方法的缺点在于:

1. 允许线程进行特权操作(开关中断), 恶意的程序可能会在进入临界区后执行死循环的操作, 导致系统崩溃. 
2. 该方法不适用于多核cpu.
3. 某些重要中断信号会丢失, 比如说IO信号.



### 使用 Test-and-set 来构建自旋锁

test-and-set是由硬件提供的原子操作指令.

```c++
int test_and_set(int* old, int new) {
    int old_val = *old;
    *old = new;
    return old;
}

struct spinlock {
    int flag;
};
void init(spinlock* lock) { lock->flag = 0; }
void lock(spinlock* lock) {
    while (test_and_set(&lock->flag, 1) == 1) ; // spin
}
void unlock(spinlock* lock) { lock->flag = 0; }
```

自旋锁一般适用于多核cpu, 如果单核cpu用自旋锁的话要求操作系统的调度方式是抢占式的.

自旋锁保证了互斥访问. 公正性无法保证. 对于性能, 多核cpu会好些, 单核则不佳.



### compare-and-swap

compare and swap 思想与 test and set 类似, 其也是硬件提供的原子操作.

```c
int compare_and_swap(int* ptr, int expected, int new) {
    int actual = *ptr;
    if (actual == expected) *ptr = new;
    return actual;
}
```



### 自旋锁的改进

对于自旋锁来说, 其循环带来的开销太大. 那么与其让线程循环, 不如让线程让出其cpu使用权, 即yield(). 可是cpu上下文切换的代价还是有点儿大. 更进一步, 可以让线程sleep



### 使用阻塞队列而非spin

```c++
struct lock {
	int flag;
    int guard;
    queue* q;
};
void lock_init(lock* l) { l->flag = 0; l->guard = 0; queue_init(l->q); }
void lock(lock* l) {
    while (test_and_set(&l->guard, 1) == 1) 
        ; // acquire guard lock by spinning
    if (l->flag == 0) {
        l->flag = 1; // lock is acquired
        l->guard = 0;
    }
    else {
        queue_add(l->q, gettid());
        l->guard = 0;
        park(); // 使线程睡眠
    }
}
void unlock(lock* l) {
    while (test_and_set(&l->guard, 1) == 1)
        ; // acquire guard lock by spinning
    if (queue_empty(l->a))
        l->flag = 0;
    else 
        unpark(queue_remove(l->q)); // 唤醒线程
    l->guard = 0;
}
```



