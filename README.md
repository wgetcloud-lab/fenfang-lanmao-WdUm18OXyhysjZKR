
在多线程开发中，经常会遇到数据同步，很多情况下用锁都是一个很好的选择。C\+\+中常用的锁主要有下面几种：



### 互斥锁（`std::mutex`）


* 这是最基本的一种锁。它用于保护共享资源，在任意时刻，最多只有一个线程可以获取该锁，从而访问被保护的资源。当一个线程获取了互斥锁后，其他试图获取该锁的线程会被阻塞，直到持有锁的线程释放它。
* 例如，在一个多线程程序中，如果多个线程需要访问和修改同一个全局变量，就可以使用互斥锁来确保在同一时间只有一个线程能够进行修改操作，避免数据竞争导致的错误结果。




```
 1 #include 
 2 #include 
 3 #include 
 4 
 5 std::mutex m;
 6 int counter = 0;
 7 
 8 void increment() {
 9     m.lock();
10     counter++;
11     std::cout << "Counter value in thread " << std::this_thread::get_id() << " is " << counter << std::endl;
12     m.unlock();
13 }
14 
15 int main() {
16     std::thread t1(increment);
17     std::thread t2(increment);
18     t1.join();
19     t2.join();
20     return 0;
21 }
```


 
### 递归互斥锁（`std::recursive_mutex`）


* 递归互斥锁允许同一个线程多次获取该锁。它内部会记录锁的获取次数，每获取一次，计数加 1，每次释放锁时，计数减 1，当计数为 0 时，锁才真正被释放，可供其他线程获取。
* 假设在一个复杂的函数调用链中，函数 A 调用函数 B，函数 B 又调用函数 A，并且这些函数都需要访问同一个受保护的资源。如果使用普通互斥锁，就会出现死锁，而递归互斥锁就可以避免这种情况，因为它允许同一线程多次获取锁。




```
 1 #include 
 2 #include 
 3 #include 
 4 
 5 std::recursive_mutex rm;
 6 
 7 void recursiveFunction(int count) {
 8     rm.lock();
 9     if (count > 0) {
10         std::cout << "Recursive call with count = " << count << std::endl;
11         recursiveFunction(count - 1);
12     }
13     rm.unlock();
14 }
15 
16 int main() {
17     std::thread t(recursiveFunction, 3);
18     t.join();
19     return 0;
20 }
```


 


### 读写锁（`std::shared_mutex`） C\+\+17开始才有


* 读写锁主要用于区分对共享资源的读操作和写操作。它有两种获取模式：共享模式（读模式）和独占模式（写模式）。
* 多个线程可以同时以共享模式获取读写锁，这意味着它们可以同时读取共享资源，而不会相互干扰。但是，当一个线程要以独占模式获取读写锁（进行写操作）时，其他线程（无论是读操作还是写操作）都不能获取该锁，直到写操作完成并释放锁。这种机制在有大量读操作和少量写操作的场景下，可以提高程序的并发性能。例如，在一个缓存系统中，多个线程可能经常读取缓存中的数据，只有在缓存数据需要更新时才会进行写操作，使用读写锁可以很好地处理这种情况。




```
 1 #include 
 2 #include 
 3 #include 
 4 #include 
 5 
 6 std::shared_mutex smtx;
 7 int shared_data = 0;
 8 
 9 void read_data() {
10     std::shared_lock lock(smtx);
11     std::cout << "Read data: " << shared_data << std::endl;
12 }
13 
14 void write_data(int new_value) {
15     std::unique_lock lock(smtx);
16     shared_data = new_value;
17     std::cout << "Wrote data: " << shared_data << std::endl;
18 }
19 
20 int main() {
21     std::vector read_threads;
22     for (int i = 0; i < 5; i++) {
23         read_threads.push_back(std::thread(read_data));
24     }
25     std::thread write_thread(write_data, 10);
26     for (auto& t : read_threads) {
27         t.join();
28     }
29     write_thread.join();
30     return 0;
31 }
```


 


### 定时互斥锁（`std::timed_mutex`）


* 定时互斥锁是`std::mutex`的扩展。除了具备`std::mutex`的基本功能外，它还允许线程在尝试获取锁时设置一个超时时间。
* 如果在规定的超时时间内无法获取锁，线程不会一直等待，而是可以执行其他操作或者返回错误信息。这在一些对时间敏感的场景中非常有用，比如在一个实时系统中，线程不能因为等待一个锁而无限期地阻塞，需要在一定时间后放弃获取并进行其他处理。




```
 1 #include 
 2 #include 
 3 #include 
 4 #include 
 5 
 6 std::timed_mutex tm;
 7 
 8 void tryLockFunction() {
 9     if (tm.try_lock_for(std::chrono::seconds(1))) {
10         std::cout << "Acquired lock" << std::endl;
11         std::this_thread::sleep_for(std::chrono::seconds(2));
12         tm.unlock();
13     } else {
14         std::cout << "Could not acquire lock in time" << std::endl;
15     }
16 }
17 
18 int main() {
19     std::thread t1(tryLockFunction);
20     std::thread t2(tryLockFunction);
21     t1.join();
22     t2.join();
23     return 0;
24 }
```


 


### 递归定时互斥锁（`std::recursive_timed_mutex`）


* 这是结合了递归互斥锁和定时互斥锁特点的一种锁。它允许同一线程多次获取锁，并且在获取锁时可以设置超时时间。
* 当一个线程多次获取这种锁后，需要释放相同次数的锁，锁才会真正被释放，并且在获取锁的过程中，如果在超时时间内无法获取，线程可以采取相应的措施。




```
 1 #include 
 2 #include 
 3 #include 
 4 #include 
 5 
 6 std::recursive_timed_mutex rtm;
 7 
 8 void recursiveTryLockFunction(int count) {
 9     if (rtm.try_lock_for(std::chrono::seconds(1))) {
10         std::cout << "Recursive acquired lock, count = " << count << std::endl;
11         if (count > 0) {
12             recursiveTryLockFunction(count - 1);
13         }
14         rtm.unlock();
15     } else {
16         std::cout << "Could not recursively acquire lock in time" << std::endl;
17     }
18 }
19 
20 int main() {
21     std::thread t(recursiveTryLockFunction, 3);
22     t.join();
23     return 0;
24 }
```


 


### 自旋锁（通常用`std::atomic_flag`实现）


* 自旋锁是一种忙等待的锁机制。当一个线程尝试获取自旋锁而锁已经被占用时，这个线程不会进入阻塞状态，而是会不断地检查（“自旋”）锁是否已经被释放。
* 自旋锁在等待时间较短的情况下可能会有比较好的性能表现，因为它避免了线程切换的开销。但是，如果等待时间过长，由于线程一直在占用 CPU 资源进行检查，会导致 CPU 资源的浪费。一般在底层代码或者对性能要求极高、等待时间预计很短的场景下使用。





```
 1 #include 
 2 #include 
 3 #include 
 4 
 5 std::atomic_flag spinLock = ATOMIC_FLAG_INIT;
 6 
 7 void criticalSection() {
 8     while (spinLock.test_and_set()) {
 9         // 自旋等待
10     }
11     std::cout << "Entered critical section" << std::endl;
12     // 临界区操作
13     spinLock.clear();
14 }
15 
16 int main() {
17     std::thread t1(criticalSection);
18     std::thread t2(criticalSection);
19     t1.join();
20     t2.join();
21     return 0;
22 }
```


 



### 条件变量（`std::condition_variable`）配合互斥锁用于同步（严格来说条件变量不是锁，但常一起用于线程同步场景）


* 条件变量本身不是一种锁，但它通常和互斥锁一起使用，用于实现线程间的同步。它可以让一个线程等待某个条件满足后再继续执行。
* 例如，一个生产者 \- 消费者模型中，消费者线程在缓冲区为空时可以使用条件变量等待，直到生产者线程生产出产品并通知消费者线程，这个过程中互斥锁用于保护缓冲区这个共享资源，条件变量用于实现线程间的通信和同步。





```
 1 #include 
 2 #include 
 3 #include 
 4 #include 
 5 #include 
 6 
 7 std::mutex mtx;
 8 std::condition_variable cv;
 9 std::queue<int> buffer;
10 const int bufferSize = 5;
11 
12 void producer() {
13     for (int i = 0; i < 10; ++i) {
14         std::unique_lock lock(mtx);
15         while (buffer.size() == bufferSize) {
16             cv.wait(lock);
17         }
18         buffer.push(i);
19         std::cout << "Produced: " << i << std::endl;
20         cv.notify_all();
21     }
22 }
23 
24 void consumer() {
25     for (int i = 0; i < 10; ++i) {
26         std::unique_lock lock(mtx);
27         while (buffer.empty()) {
28             cv.wait(lock);
29         }
30         int data = buffer.front();
31         buffer.pop();
32         std::cout << "Consumed: " << data << std::endl;
33         cv.notify_all();
34     }
35 }
36 
37 int main() {
38     std::thread producerThread(producer);
39     std::thread consumerThread(consumer);
40     producerThread.join();
41     consumerThread.join();
42     return 0;
43 }
```


 





 本博客参考[悠兔机场](https://xinnongbo.com)。转载请注明出处！
