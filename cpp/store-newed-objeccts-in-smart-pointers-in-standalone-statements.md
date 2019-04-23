# 单独列一条语句来使用智能指针
假设我们有一个函数 `priority()` 返回处理优先级，还有一个函数 `processWidget()` 用来处理动态分配的对象 `Widget` 及优先级：
```c++
    int priority();
    void processWidget(std::tr1::shared_ptr<Widget> pw, int priority);
```

以下代码可以编译通过：
```c++
    processWidget(std::tr1::shared_ptr<Widget>(new Widget), priority());
```
但令人诧异的是，即使我们在每一处都使用对象管理资源，这样调用仍可能导致资源泄漏。接下来，我们解释以下这是怎么回事。

。。第二个参数。。。但第一个实参 `std::tr1::shared_ptr<Widget>(new Widget)` 含有两部分：
+ 执行表达式 `new Widget`；
+ 调用 `tr1::shared_ptr` 的构造函数。

在调用方法 `processWidget` 之前，编译器一定做三件事在生成代码时：
+ 调用方法 `priority`
+ 执行 `new Widget`
+ 调用 `tr1::shared_ptr` 的构造函数。

C++ 编译器有余地决定以上三件事发生的次序。（这和其他语言例如 Java 和 C# 总是以特定次序操作函数参数的工作方式不同）。其中表达式 `new Widget` 一定在调用 `tr1::shared_ptr` 的构造函数之前执行，因为 `new Widget` 要作为实参传给 `tr1::shared_ptr` 的构造函数，然而方法 `priority()` 的调用次序可以放在第一位、第二位或者第三位。如果编译选择将其放在第二位执行（可能为生成更高效的代码），我们就会有如下的操作次序：
+ 执行 `new Widget`
+ 调用方法 `priority`
+ 调用 `tr1::shared_ptr` 的构造函数。

然而，如果调用方法 `priority()` 时抛出了异常。在那样的情况下，`new Widget` 返回的对象指针将丢失，因为



要避免这种问题的方法很简单：单独列一条语句来创建 `Widget` 并且将其置入智能指针，之后再下一条语句将智能指针传给方法 `processWidge`：
```c++
    std::tr1::shared_ptr<Widget> pw(new Widget); // 在单独的语句里将新建对象置入智能指针
    processWidget(pw, priority());               // 此调用将不会导致泄露
```
这样做很凑效，因为编译器将没有余地改变操作次序。。。在这个改进语句，

# 牢记
+ 单独列一条语句来使用智能指针。如果不这样做，可能在抛出异常时导致的内存泄露。