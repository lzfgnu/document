# Ostep_cp26 并发简介

### 使用线程的两点原因

1. 提高程序的并行度, 使程序运行的更快

2. 防止程序因IO而被操作系统block

   如果程序是多线程的, 当它发起IO操作的时候, 可以由一个线程去处理IO时间, 其他的线程继续运行其他的代码(如果有的话). 

   

### 线程共享变量示例

```c
static volatile int cnt = 0;

void* thread_entry(void* arg) {
    printf("%s: begin\n", (char*)arg);
    for (int i = 0; i < 1e7; ++i) ++cnt;
    printf("%s: end\n", (char*)arg);
    return NULL;
}
int main() {
    pthread_t p1, p2;
    pthread_create(&p1, NULL, thread_entry, "A");
    pthread_create(&p2, NULL, thread_entry, "B");
    pthread_join(p1, NULL);
    pthread_join(p2, NULL);
    return 0;
}
```

这段代码并没有对共享变量加锁. 所以面临数据竞争的问题. 其原因是因为++cnt的汇编指令不止一条并且也不是原子操作. 其可能的指令如下

```assembly
mov 0x8049a1c, %eax
add $0x1, %eax
mov %eax, 0x8049a1c
```



