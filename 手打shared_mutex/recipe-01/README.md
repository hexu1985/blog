# 手打shared_mutex（一）

## shared_mutex接口简介

shared_mutex 类是一个同步原语，可用于保护共享数据不被多个线程同时访问。

与便于独占访问的其他互斥类型不同，shared_mutex 拥有二个访问级别：

- 共享 - 多个线程能共享同一互斥的所有权。

- 独占性 - 仅一个线程能占有互斥。

shared_mutex类的公开方法包括：

![shared_mutex public member functions](../png/shared_mutex_public_member_functions.png)

std::shared_mutex是在C++17标准中引入的，std::shared_mutex的更完整描述可以在[cppreference.com](https://en.cppreference.com/w/cpp/thread/shared_mutex)网页上找到。

## shared_mutex实现原理

对于非C++标准来说，shared_mutex的更容易理解的名称是**读写锁**（read-write lock）。

相比于**读写锁**，更基础的是**互斥锁**，所以我们就从互斥锁说起。

**互斥锁**会把试图进入**临界区**的所有其他线程都阻塞住。该临界区通常涉及对由这些线程共享的一个或多个数据的访问或更新。然而有时候我们可以在**读**某个数据与**修改**某个数据之间作区分。这也是使用**读写锁**的场景条件之一。

对于在获取读写锁用于读或获取读写锁用于写之间作区别，规则如下：

(1) 只要没有线程持有某个给定的读写锁用于写，那么任意数目的线程可以持有该读写锁用于读。

(2) 仅当没有线程持有某个给定的读写锁用于读或用于写时，才能分配给读写锁用于写。

