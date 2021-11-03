# C++ 标准库中的allocator是多余的

我认为 C++ 的 allocator 是依赖注入的一次失败的尝试。

C/C++ 里的内存分配和释放是个重要的事情，我同意，在写 library 的时候，除了默认使用 malloc/free，还应该允许用户指定使用内存分配的函数。用现在的话说，如果 library 依赖于内存分配与释放，就应该允许用户注入这种依赖。我看到有些 C library 是支持这个的，可以在初始化时传入两个函数指针，指向内存分配和释放的函数。

问题是，allocator 是模板参数，而不是构造函数的参数。这意味着

1. 由于不能从构造函数传入 allocator，那么每种类型的 allocator 必须是全局唯一的（Singleton）。无论 SGI 的内存池（称为 PoolAlloc），还是简单的 new wrapper（称为 NewAlloc）都只从一个地方 (region) 搞到内存，这大大限制了其使用。 （补注：这是 SGI STL 的限制，标准 C++ 运行从构造函数传入 allocator，后面的论述依然成立。）


2. allocator 是 vector 类型的一部分，vector<string, PoolAlloc> 和 vector<string, NewAlloc> 是两个类型，不可相互替换。这不仅暴露了实现，还暴露到了类型上，恐怕没有比这更糟糕的了。

下面举例说明，

对于 1，假设我有一个任务（假设是 parse），需要分配很多小块内存，总容量不超过 20M。为了防止内存泄露及避免内存碎片，我希望在任务开始时，先从系统拿到 20M 内存，供这个任务使用（parse 里分配内存只需要改一个指针，释放内存是空操作），等任务完成后，我一次性释放这 20M 内存，这样既高效又安全。然而 C++ 的 allocator 并不能帮我实现这一点，因为它是全局的。我不能替换全局的 allocator，因为那会影响其他线程。也不能在运行时指定某个 vector<string> 用哪种 allocator，因为类型是编译时确定的。

对于 2，如果我想写一个普通的以 vector<string> 为参数的函数，这不难


void process(vector<string>& records);

由于 vector<string, PoolAlloc> 和 vector<string, NewAlloc > 类型不同，process 只能接受一种。
但这完全没道理，我不过想访问一个 vector<string>，根本不关心它的内存是怎么分配的，却被什么鬼东西 allocator 挡在了门外。


我要么提供重载：
void process(vector<string, NewAlloc>& records);
void process(vector<string, PoolAlloc>& records);

要么改写成模板：
template<typename Alloc>
void process(vector<string, Alloc>& records);

//（同理可知，如果在一个程序里使用多种 allocator，那么所有涉及标准库容器的用户函数都必须改写为函数模板）

无论哪种 "解决办法" 都会导致代码膨胀，而且给标准库的使用者带来完全不必要的负担。

更糟糕的是，allocator 给程序库的作者也带来了不必要的负担。如果想把 process(vector<string>& records) 放到某个 library 中，那么为了适应不同的 allocator，必须把函数定义放在头文件里（因为这是个函数模板）。明明是针对一个固定类型（vector of string）的函数，却不得不写成函数模板，把实现细节暴露在头文件里，让每个用户都去编译一遍，这真是完全没道理。

根据以上的分析，基本上不可能在一个程序里混用多种 allocator，既然一个程序只能有一种 allocator，那为什么还要放到每个容器的模板参数里呢？提供一个全局的钩子不就行了嘛？

相反，shared_ptr 就只有一个模板参数 T，而他同样可以指定 allocator---- 在构造时传入。

现在看来，vector（以及其他标准库容器）与其增加一个 Alloc 模板参数，不如在构造时传入两个函数指针，一个 allocate，一个 deallocate，定制的效果也一样。只不过这么做会让标准委员会的人觉得不够 GP，很可能被拍掉。

总而言之，allocator 并不能达到精确控制（定制）内存分配释放的目的，虽然它名以上是为此而生的。虽然在历史上可能有过正面效果（封装 far / near pointer），但现在它无疑就是个累赘。就跟 IOStream 的 Locale/Facet ，auto_ptr 和 valarray 一样，成为 C++ 标准一直要背负的历史包袱。

---
1. 见书 P510
2. 原文链接：https://blog.csdn.net/Solstice/article/details/4401382
