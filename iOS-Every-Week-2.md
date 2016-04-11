# iOS Every Week（2）

这周的主题是**Run Loop**，请按要求完成以下知识点的学习：

## Key Points

- Run Loop对外接口的结构和作用
- Run Loop的内部逻辑
- 理解Mach Port
- Run Loop的迭代执行顺序
- Apple使用Run Loop实现的功能
- 如何在实际操作中去使用Run Loop

## Questions

- 如何让Run Loop挂起or唤醒？
- 为什么NSTimer完全依赖于Run Loop？GCD的计时器和Run Loop有关系吗？
- 主线程的函数是通过Run Loop哪些函数调起的？
- source0和source1有什么区别？
- 解释一下Run Loop的四种mode。
- 一个thread能对应几个Run Loop？
- Observer的CFRunLoopActivity的枚举分别代表了什么状态？
- 从Run Loop角度解释一下autorelease对象在何时释放？