Redis中的键都是string类型

## SDS

    Redis字符类型是自定义类型SDS（Simple Dynamic String）。

### 1 特点

1. SDS是二进制安全的数据结构

2. 提供了内存预分配，避免了频繁的内存扩容

3. 兼容c语言函数库


