# Ostep_cp31 信号量

### 定义

信号量是与一个整数关联的对象, 可以用两种方法对其操作. 一个是wait, 一个是post. 信号量的初始值决定了该信号量对象的行为. 

```c
int sem_wait(sem_t* s) {
    // 将信号量数值减 1
    // 如果信号量值变为负数, 则等待
}
int sem_post(sem_t* s) {
    // 将信号量数值加 1
    // 唤醒一个线程
}
sem_t m;
sem_init(&m, 0, X); // X 是信号量的初始值
```



### 二元信号量(锁)

如果信号量的初始值为1, 那么其如同互斥锁(mutex lock)



### 信号量与调度

可以使用信号量来实现thread_join, 只需令信号量初始值为0即可.

```c
sem_t s;

void* child(void* arg) {
    printf("child\n");
    sem_post(&s);
    return NULL;
}
int main() {
	sem_init(&s, 0, 0);
    printf("praent: begin\n");
    pthread_t c;
    pthread_create(&c, NULL, child, NULL);
    sem_wait(&s);
    printf("parent: end\n");
    return 0;
}
```



### 信号量与生产者/消费者问题

信号量根据不同值等同于互斥锁和条件变量,  因此也就能解决生产者/消费者问题

```c
sem_t empty, full, mutex;
void* producer(void* arg) {
    for (int i = 0; i < loops; ++i) {
        sem_wait(&empty);  // 1
        sem_wait(&mutex);  // 2
        put(i);
        sem_post(&mutex);
        sem_post(&full);
    }
}
void* consumer(void* arg) {
    for (int i = 0; i < loops; ++i) {
        sem_wait(&full);
        sem_wait(&mutex);
        int tmp = get();
        sem_post(&mutex);
        sem_post(&empty);
        printf("%d\n", tmp);
    }
}
int main() {
    sem_init(&empty, 0, MAX);
    sem_init(&full, 0, 0);
    sem_init(&mutex, 0, 1);
    // ...
}
```

##### 为什么1, 2 处的顺序不能颠倒?

因为会引发死锁问题. 假设线程先获取锁, 而此时buffer满了, 那么它将进入sleep状态. 可是它又把持着锁, 使得其他线程因无法获取这把锁而等待. 最终程序僵死.



### 读写锁

```cpp
class RWLock {
public:
    RWLock() {
        readers = 0;
        sem_init(&lock, 0, 1);
        sem_init(&write_lock, 0, 1);
    }
    ~RWLock() { /* ... */ }
    void ReadLock() {
        sem_wait(&lock);
        ++readers;
        if (readers == 1) 
            sem_wait(&write_lock);  // 1
        sem_post(&lock);
    }
    void ReadUnlock() {
        sem_wait(&lock);
        --readers;
        if (readers == 0) 
            sem_post(&write_lock);
        sem_post(&lock);
    }
    void WriteLock() {
        sem_wait(&write_lock);
    }
    void WriteUnlock() {
        sem_post(&write_lock);
    }
private:
    sem_t lock;
    sem_t write_lock;
    int readers;
};
```

对于读操作来说, 先将write_lock给把持住, 这样写操作就无法进行了, 从而reader threads 可以放心大胆的读. 如果在读之前, 已经有线程正进行写操作, 那么该进行读操作的线程就在1处等待, 同时它把持着lock, 使得其他想要进行读操作的线程也必须等待.

