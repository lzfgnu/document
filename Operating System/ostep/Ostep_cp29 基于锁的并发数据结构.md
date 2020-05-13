# Ostep_cp29 基于锁的并发数据结构

### 并发计数器

1. 简单但是伸缩性差的实现

   这种实现简单并且正确, 但是随着线程数量的增多, 其性能也会随之下降, 而且下降的还挺严重的.

   ```c++
   class ConcurrentCounter {
   public:
       ConcurrentCounter() {
       	val = 0;
      		pthread_mutex_init(&lock, nullptr);
   	}
       
       ~ConcurrentCounter() {
           // ...
       }
       
       void Increment() {
           pthread_mutex_lock(&lock);
           ++val;
           pthread_mutex_unlock(&lock);
       }
       
       void Decrement() {
           pthread_mutex_lock(&lock);
           --val;
           pthread_mutex_unlock(&lock);
       }
       
       int Get() {
           pthread_mutex_lock(&lock);
           int rc = val;
           pthread_mutex_lock(&lock);
           return rc;
       }
   private:
       int val;
       pthread_mutex_t lock;
   };
   ```

2. 伸缩性较好的实现方式(sloopy counter)

   这种实现把计数器分为全局计数器和本地计数器. 每个cpu享有其本地的计数器并且共享一个唯一的全局计数器. 比如说, 一个四核cpu包含4个本地计数器和一个全局计数器. 除此之外, 还有本地锁和全局锁. 因为每个cpu有其私有的计数器, 所以这些cpu上的线程在更新本地计数器的时候不会发生竞争.

   为了保证全局计数器的值是最新的. 这些本地计数器的值必须被周期性的传送给全局计数器以使其更新. 至于多久跟新一次则由周期S确定. S越小, 计数器的性能越接近第一种实现方式, 因此全局计数器的值也来的更精确. S越大, 锁的性能越优, 但是全局计数器的值可能会不准确.

   ```c++
   class ConcurrentCounter {
   public:
       ConcurrentCounter(int threadhold = 5) {
           this->threadhold = threadhold;
           global_val = 0;
           pthread_mutex_init(&global_lock, nullptr);
           for (int i = 0; i < num_cpus; ++i) {
               local_val[i] = 0;
               pthread_mutex_init(&local_lock[i], nullptr);
           }
       }
       ~ConcurrentCounter() { /* ... */ }
       
       void Update(int thread_id, int amt) {
           int cpu = thread_id % num_cpus;
           pthread_mutex_lock(&local_lock[cpu]);
           local_val[cpu] += amt;
           if (local_val[cpu] >= threadhold) {
               pthread_mutex_lock(&global_lock);
               global_val += local_val[cpu];
               pthread_mutex_unlock(&global_lock);
               local_val[cpu] = 0;
           }
           pthread_mutex_unlock(&local_lock[cpu]);
       }
        
       int Get() {
           pthread_mutex_lock(&global_lock);
           int val = global_val;
           pthread_mutex_unlock(&global_lock);
           return val;
       }
   private:
       int global_val;
       pthread_mutex_t global_lock;
       int local_val[num_cpus];
       pthread_mutex_t local_lock[num_cpus];
       int threadhold; // S
   };
   ```



### 并发链表

1. 简单但是伸缩性差的实现

   ```c++
   class Node {
   public:
       int key;
       Node* next;
   };
   
   class ConcurrentLinkedList {
   public:
       ConcurrentLinkedList() {
           head = nullptr;
           pthread_mutex_init(&lock, nullptr);
       }
       
       int Insert(int key) {
           Node* new_node = new Node();
           if (new_node == nullptr) { /* throw */ }
           new_node->key = key;
           
           pthread_mutex_lock(&lock);
           new_node->next = head;
           head = new_node;
           pthread_mutex_unlock(&lock);
           return 0;
       }
       
   private:
       Node* head;
       pthread_mutex_t lock;
   };
   ```

2. 手把手加锁

   这种实现方式是每个node都包含一把锁, 当遍历链表的时候, 先获取下一个结点的锁, 然后释放当前结点的锁. 这种做法的优点是并发度高, 但是其性能却说不上优, 因为每个结点都涉及申请/释放锁, 其代价较高.



### 并发队列

1. 平凡实现

   与上两种数据结构第一种类似, 即用一把大锁锁住临界区

2. 头尾锁

   这种实现使用了哨兵结点, 使得入队和出队操作可以分离. 并且使用了头尾锁以提高并发度.

   ```c++
   class Node {
   public:
       int val;
       Node* next;
   };
   
   class ConcurrentQueue {
   public:
       ConcurrentQueue() {
           Node* tmp = new Node();
           tmp->next = nullptr;
           head = tail = tmp;
           pthread_mutex_init(&head_lock);
           pthread_mutex_init(&tail_lock);
       }
       
       ~ConcurrentQueue() { /* ... */ }   
       
       void Enque(int val) {
           Node* tmp = new Node();
           tmp->val = val;
           tmp->next = nullptr;
           
           pthread_mutex_lock(&tail_lock);
           tail->next = tmp;
           tail = tmp;
           pthread_mutex_unlock(&tail_lock);
       }
       
       int Deque(int* res) {
           pthread_mutex_lock(&head_lock);
           Node* tmp = head;
           Node* new_head = tmp->next;
           if (new_head == nullptr) {
               pthread_mutex_unlock(&head_lock);
               return -1;
           }
           *val = new_head->val;
           head = new_head;
           pthread_mutex_unlock(&head_lock);
           free(tmp);
           return 0;
       }
   private:
       Node* head;
       Node* tail;
       pthread_mutex_t head_lock;
       pthread_mutex_t tail_lock;
   };
   ```



### 并发哈希表

1. 每桶每锁

   每个hash桶都分配一把锁

