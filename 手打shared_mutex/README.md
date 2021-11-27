手打shared_mutex
================


### shared_mutex简介

shared_mutex 类是一个同步原语，可用于保护共享数据不被多个线程同时访问。

与便于独占访问的其他互斥类型不同，shared_mutex 拥有二个访问级别：

- 共享 - 多个线程能共享同一互斥的所有权。

- 独占性 - 仅一个线程能占有互斥。

shared_mutex类的公开方法包括：

![shared_mutex public member functions](png/shared_mutex_public_member_functions.png)

std::shared_mutex是在C++17标准中引入的，std::shared_mutex的更完整描述可以在[cppreference.com](https://en.cppreference.com/w/cpp/thread/shared_mutex)网页上找到。

***

### shared_mutex语义

对于非C++标准来说，shared_mutex的更容易理解的名称是**读写锁**（read-write lock）。

相比于**读写锁**，更基础的是**互斥锁**，所以我们先从互斥锁说起（互斥锁在C++标准中的名称是std::mutex）。

**互斥锁**会把试图进入**临界区**的所有其他线程都阻塞住。该临界区通常涉及对由这些线程共享的一个或多个数据的访问或更新。然而有时候我们可以在**读**某个数据与**修改**某个数据之间作区分。这也是使用**读写锁**的场景条件之一。

对于在获取读写锁用于读或获取读写锁用于写之间的区别，规则如下：
- 只要没有线程持有某个给定的读写锁用于写，那么任意数目的线程可以持有该读写锁用于读。
- 仅当没有线程持有某个给定的读写锁用于读或用于写时，才能分配给读写锁用于写。

换一种说法就是，只要没有线程在修改某个给定的数据，那么任意数目的线程都可以拥有该数据的读访问权。仅当没有其他线程在读或修改某个给定的数据时，当前线程才可以修改它。

某些应用中读数据比修改数据频繁，这些应用可以从改用读写锁代替互斥锁中获益。任意给定时刻允许多个读出者存在提供了更高的并发度，同时在某个写入者修改数据期间保护该数据，以免任何其他读出者或写入者的干扰。这也是判断是否应该使用**读写锁**的依据之一。

这种对于某个给定资源的共享访问也称为共享-独占（shared-exclusive）上锁（这就是shared_mutex中shared的由来），其中获取一个读写锁用于读称为**共享锁**（shared lock），对应的就是shared_mutex::lock_shared方法，获取一个读写锁用于写称为**独占锁**（exclusive lock），对应的就是shared_mutex::lock方法。有人可能有疑问，为啥独占锁对应的方法是shared_mutex::lock，而不是shared_mutex::lock_unique之类的呢？其实原因是为了跟std::mutex的lock方法名一致，而这样做的原因是为了让std::lock_guard和std::unique_lock等模板类能复用在shared_mutex上，具体细节这里就不展开了。

***

### shared_mutex实现

接下来，我们将自己动手实现一个shared_mutex。

##### 1. shared_mutex类的数据结构

shared_mutex包含如下成员变量：
```cpp
class shared_mutex {
private:
    std::mutex              mutex;
    std::condition_variable read;       // wait for read
    std::condition_variable write;      // wait for write
    int                     r_active;   // readers active
    int                     w_active;   // writer active
    int                     r_wait;     // readers waiting
    int                     w_wait;     // writers waiting
// ...
};
```
- 用于序列化成员变量存取的互斥量（mutex）。
- 两个独立的条件变量，一个用于等待读操作（read），一个用于等待写操作（write）。
- 为了能够判断条件变量上是否有等待线程，我们将保存活跃的读线程数（r_active）和一个指示活跃的写线程数（w_active）的标志。
- 我们也保留了等待读操作的线程数（r_wait）和等待写操作的线程数（w_wait）。

这里w_active虽然是int类型，其实是表示一个bool标志，因为只能有一个”活动的写线程“（持有写锁），但考虑到内存对齐，这里使用bool还是int，其实是没差别的。

##### 2. shared_mutex类的构造与析构。

首先是构造函数：
```cpp
shared_mutex::shared_mutex():
    r_active(0), w_active(0), r_wait(0), w_wait(0)
{
}
```
并没有什么需要特别强调的，mutex、read、write这三个成员变量会调用默认构造函数，shared_mutex对象构造完成后，处于未加锁的状态。
	
然后是析构函数：

```cpp
shared_mutex::~shared_mutex()
{
    assert(r_active == 0);
    assert(w_active == 0);
    assert(r_wait == 0);
    assert(w_wait == 0);
}
```
更是没什么好说的，只是加了一些断言而已。

##### 4. shared_mutex类的获取/释放共享锁（读锁）的相关方法。

首先是lock_shared方法，为读操作获取共享锁（读锁）。
```cpp
void shared_mutex::lock_shared()
{
    std::unique_lock<std::mutex> lock(mutex);
    if (w_active) {
        r_wait++;
        while (w_active) {
            read.wait(lock);
        }
        r_wait--;
    }
    r_active++;
}
```
- 如果当前没有写线程是活动的，那么我们就登记r_active成员变量，更新活动的读者线程数+1，并从lock_shared方法返回，表示获取共享锁成功。
- 如果一个写线程当前是活动的（w_active非0），我们就登记r_wait成员变量，更新等待读线程的数量+1，然后wait在read条件变量上（直到写线程释放了锁，并通过read条件变量唤醒了当前线程）。
- 另外，为了简化实现，这里并没考虑线程wait在read条件变量时，线程被cancel的情况，这种情况下r_wait并不会-1，从而造成数据不一致。对于这种情况的处理，可以参考相关书籍，这里就不在赘述了。

然后是try_lock_shared方法：

```cpp
bool shared_mutex::try_lock_shared()
{
    std::lock_guard<std::mutex> lock(mutex);
    if (w_active) {
        return false;
    } else {
        r_active++;
        return true;
    }
}
```
逻辑上和lock_shared几乎相同，除了：
- 当有一个写线程活动时，它将直接返回false，而不是block在read条件变量上。
- 如果获取共享锁（读锁）成功，返回的true。

最后是unlock_shared方法：

