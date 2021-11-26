# 手打shared_mutex（一）

## shared_mutex简介

shared_mutex 类是一个同步原语，可用于保护共享数据不被多个线程同时访问。

与便于独占访问的其他互斥类型不同，shared_mutex 拥有二个访问级别：

- 共享 - 多个线程能共享同一互斥的所有权。

- 独占性 - 仅一个线程能占有互斥。

shared_mutex类的公开方法包括：

![shared_mutex public member functions](../png/shared_mutex_public_member_functions.png)

