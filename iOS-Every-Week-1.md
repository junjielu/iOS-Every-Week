# iOS Every Week（1）

这周的主题是**内存管理**，请按要求完成以下知识点的学习：

## Key Points

- 引用计数的工作原理
- assign strong weak unsafe_unretained copy 分别拥有怎样的内存管理策略
- 常见的内存泄露场景和处理方法
- autorelease的机制
- autorelease pool的机制和生命周期

## Questions

- 什么是野指针？如何防止野指针的出现？
- autorelease对象什么时候释放？如何手动干预autorelease对象的释放时间？
- NSThread、NSRunLoop 和 NSAutoreleasePool三者的关系是什么？
- 延迟释放对象有什么好处？
- dealloc方法中需要处理什么事情？
- 将引用设为weak可以避免循环引用，为什么unsafe_unretain不行？
- ARC下对于self的管理是怎样的？
- main函数中的autoreleasepool起什么作用？能删去嘛？
- 如何去做内存优化？