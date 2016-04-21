# Block - 陆俊杰

## Key Points

### block的内部结构

新建一个 `BlockTest.c` 文件实现一个简单的block：

```
int main()
{
    ^{ printf("This is a block!"); } ();
    return 0;
}
```
在命令行中输入 `clang -rewrite-objc BlockTest.c` 将代码转换为C++代码:

``` 
struct __block_impl {
    void *isa;
    int Flags;
    int Reserved;
    void *FuncPtr;
};


struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
 printf("This is a block!"); }

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};

int main()
{
    ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA)) ();
    return 0;
}
```

让我们来分析一下上面的关键代码。

对应之前block中的代码 `printf("This is a block!");` 在转换后变成了

```
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  printf("This is a block!"); 
}
```

忽略构造函数，这里的 `__main_block_impl_0` 的结构如下所示:

```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
};
```

很明显， `__main_block_impl_0`  是由 `__block_impl` 和 `__main_block_desc_0` 构成。对于 `__block_impl` ，其结构如下：

```
struct __block_impl {
    void *isa;
    int Flags;
    int Reserved;
    void *FuncPtr;
};
```

对于 `__main_block_desc_0` ，结构如下：

```
static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
}
```

我们观察一下构造函数：

```
__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
```

这里将所有结构体成员进行了初始化。了解了结构之后，我们可以看看构造函数是如何调用的。

```
((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA)) ();
```

这里我们做个简化:

```
__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA)) ();
```

这样就比较好理解了，源码中的block是 `__main_block_impl_0 ` 结构体类型的自动变量。而 `__main_block_impl_0 ` 结构体相当于基于 `objc_object` 结构体的类对象的结构体，因此，block是 `Objective - C` 的对象。

### block的生命周期

通常来说，对象是在堆上生成的，但是通过Clang转换的内部实现我们可以看到block是在栈上生成的。

对于block来说，到底是在堆上生成还是在栈上生成是根据其isa指针判定的。

类           				 | 存储域
----------------------- | -------------
_NSConcreteStackBlock   | 栈
_NSConcreteGlobalBlock  | .data 区
_NSConcreteMallacBlock  | 堆

简单介绍一下这三个类：

- **_NSConcreteGlobalBlock** ： 没有引用外部变量的block
- **_NSConcreteStackBlock** ： 引用了外部变量的block
- **_NSConcreteMallacBlock** ： 被拷贝的block

> 这里我们会提出疑问：之前示例的block并没有引用外部变量，为什么isa指针却指向 `_NSConcreteStackBlock` 呢？
> 
> 原因其实很简单，由于Clang的改写和 `LLVM` 的编译并不是完全相同的，所以 isa 指向的是 `_NSConcreteStackBlock` 。但在 `LLVM` 的编译结果中，开启 ARC 时，block 应该是 `_NSConcreteGlobalBlock` 类型。

### block在ARC和MRC下的区别

对于 `_NSConcreteStackBlock` 来说，在开启ARC时，会通过 `objc_retainBlock()` 函数将block拷贝到堆内存，并作为一个 `Objective-C` 对象放入 `Autoreleasepool` 里面，从而保证了返回后的block仍然可以正确执行，这个操作叫做 `自动拷贝` 。

> 很多博客介绍说，在开启ARC时，只会有 `_NSConcreteGlobalBlock` 和 `_NSConcreteMallocBlock` 类型的block，然而这种观点是错误的。导致这个观点错误的原因是我们使用block的时候常常会使用一个block变量进行赋值，比如：

> ```
> void (^blk)(void) = ^{
  printf("Hello, World!");
> }
> ```

> 在开启ARC下，赋值操作默认为强引用，而对于block来说 `_NSConcreteGlobalBlock` 的赋值操作会被默认为copy，这时就生成了 `_NSConcreteMallocBlock` 。

### `__block`关键字的作用

对于一般的外部变量的截获，block的处理方法是copy一份进来。而对于 `__block` 修饰的变量，block的处理方法是copy其指针，在内部进行retain使其引用计数递增保证存活。

对于 `__block` 修饰的外部变量，block是通过创建一个结构体存储的：

```
struct __Block_byref_intValue_0
{
    void *__isa; 
    __Block_byref_intValue_0 *__forwarding;
    int __flags; 
    int __size; 
    int intValue; 
};
```

访问 `__block` 变量内部的成员变量是通过访问 `__forwarding` 指针完成的，在block从栈上拷贝到堆上时，栈上的 `__forwarding` 指针会指向堆上的 block 结构体实例

## Questions

### 请用代码写出block的数据结构

> ```
> struct Block_descriptor {
    unsigned long int reserved;
    unsigned long int size;
    void (*copy)(void *dst, void *src);
    void (*dispose)(void *);
> };

> struct Block_layout {
    void *isa;
    int flags;
    int reserved;
    void (*invoke)(void *, ...);
    struct Block_descriptor *descriptor;
> };
> ```
> ——From Matt Galloway


### block的isa指针指向谁？

指向 `_NSConcreteStackBlock` 或 `_NSConcreteGlobalBlock` 或`_NSConcreteMallacBlock`  


### 为什么在block中对外部变量进行操作是OK的，而赋值是被禁止的？

没有 `__block` 修饰的情况下，block对外部变量的截获只是做了 _值传递_ 因此没有办法去改变其引用，只能做值的修改。

### block和_block变量的存储域是分配到哪里的？

block和 `__block` 变量都在栈上生成，只有当block为引入外部变量的情况下才会分配到.data区。大多数情况下，编译器会将栈上的block自动拷贝到堆上，只有当block入参时，编译器不会自动调用 copy 方法。

> 一些特殊情况会将block进行拷贝，比如：
> 
>  - block作为函数返回值返回
>  - block被赋值给 `__strong id` 类型的对象或 block 的成员变量
>  - block作为参数被传入方法名带有 usingBlock 的 `Cocoa Framework` 方法或 GCD 的 API 

当block从栈上拷贝到堆上时，为了保证能正确访问栈上的 `__block` 变量，会将栈上的 `__forwarding` 指针指向了堆上的 block 结构体实例。

### block是如何截获对象的？

无 `__block` 修饰：

![变量复制](http://blog.devtang.com/images/block-capture-1.jpg)

有 `__block` 修饰：

![指针引用](http://blog.devtang.com/images/block-capture-2.jpg)