# 单独列一条语句来使用智能指针
假设我们有一个函数 `priority()` 返回处理优先级，还有一个函数 `processWidget()` 用来处理动态分配的对象 `Widget` 及优先级：
```c++
    int priority();
    void processWidget(std::tr1::shared_ptr<Widget> pw, int priority);
```

谨记用对象来管理资源（参见 13），`processWidget` 使用了智能指针（即这里的 `tr1::shared_ptr`）来管理动态分配的对象 `Widget`。

现在考虑调用一下 `processWidget`：
```c++
    processWidget(new Widget, priority());
```

等等！不是那样调用，那样不可以通过编译的。`tr1::shared_ptr` 的构造函数需要显式的传入原始指针，所以不能够从 `new Widget` 到智能指针 `tr1::shared_ptr` 的隐式类型转换。

以下代码就可以编译通过：
```c++
    processWidget(std::tr1::shared_ptr<Widget>(new Widget), priority());
```
但令人诧异的是，即使我们在每一处都使用对象管理资源，这样调用仍可能导致资源泄漏。接下来，我们解释以下这是怎么回事。

当编译器分析到函数调用 `processWidget` 之前就一定会分析到传入的所有参数。第二个实参是函数调用 `priority()`，但第一个实参 `std::tr1::shared_ptr<Widget>(new Widget)` 含有两部分：
+ 执行表达式 `new Widget`；
+ 调用 `tr1::shared_ptr` 的构造函数。

在调用方法 `processWidget` 之前，编译器在编译期间一定会做以下三件事：
+ 调用方法 `priority`
+ 执行 `new Widget`
+ 调用 `tr1::shared_ptr` 的构造函数。

C++ 编译器有余地决定以上三件事发生的次序。（这和其他语言例如 Java 和 C# 总是以特定次序操作函数参数的工作方式不同）。其中表达式 `new Widget` 一定在调用 `tr1::shared_ptr` 的构造函数之前执行，因为 `new Widget` 要作为实参传给 `tr1::shared_ptr` 的构造函数，然而方法 `priority()` 的调用次序可以放在第一位、第二位或者第三位。如果编译选择将其放在第二位执行（可能为生成更高效的代码），我们就会有如下的操作次序：
+ 执行 `new Widget`
+ 调用方法 `priority`
+ 调用 `tr1::shared_ptr` 的构造函数。

然而，如果调用方法 `priority()` 时抛出了异常。在那样的情况下，`new Widget` 返回的对象指针将丢失，因为它并不是我们期待的方式置入智能指针，而这将会导致资源泄露。因为在创建资源和将资源转换到资源管理对象时可能发生异常干扰，从而导致在调用 `processWidget` 时发生资源泄露。

要避免这种问题的方法很简单：单独列一条语句来创建 `Widget` 并且将其置入智能指针，之后再下一条语句将智能指针传给方法 `processWidge`：
```c++
    std::tr1::shared_ptr<Widget> pw(new Widget); // 在单独的语句里将新建对象置入智能指针
    processWidget(pw, priority());               // 此调用将不会导致泄露
```
这样做很凑效，因为编译器将没有余地改变操作次序。在这条更改后的代码，创建对象的表达式 `new Widget` 和智能指针的构造函数调用结束后，才调用 `priority()`，即它们不一条语句里，所以编译器是不可能允许调用 `priority()` 发生在两者之间的。

# 牢记
+ 单独列一条语句来使用智能指针。如果不这样做，可能在抛出异常时导致的内存泄露。
