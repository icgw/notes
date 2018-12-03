# C++ 默默编写并调用了哪些函数？
什么时候 **空类** 不再为空？当你的 C++ 代码段中调用它的时候。即使你没有声明 **拷贝构造函数**、 **拷贝赋值操作符** 、 **析构函数** ，编译器也会声明自己版本的这些函数。即使没有 **构造函数** ，它也能给你声明 **默认构造函数** 。这些函数都是 `public` 且 `inline` 的（参见30）。

因此，如果你写下此段代码：
```c++
class Empty {};
```

这就好比如下代码：
```c++
class Empty {
    public:
        Empty() {...}                               // 默认构造函数
        Empty(const Empty& rhs) {...}               // 拷贝构造函数
        ~Empty() {...}                              // 析构函数

        Empty& operator=(const Empty& rhs) {...}    // 拷贝赋值操作符
};
```
仅在需要这些函数时，编译器则生成它们。如下述代码，编译器隐式地生成所需的函数：
```c++
Empty e1;       // 默认构造函数
                // 析构函数

Empty e2(e1);   // 拷贝构造函数
e2 = e1;        // 拷贝赋值操作符
```

编译器为你生成的这些函数有什么用处呢？默认构造函数、析构函数主要是给编译器处理隐含操作的空间，比如调用基类的构造函数和析构函数、非静态成员的数据。注意，生成的析构函数是非虚的（参见7），除非类的基类声明了虚析构函数（这种情况下函数的虚属性来自基类）。

至于拷贝构造函数和拷贝赋值操作符，编译器生成的版本只是简单复制源对象的每一份非静态成员的数据到目标对象。

考虑下述示例：
```c++
template<typename T>
class NamedObject {
    public:
        NamedObject(const char *name, const T& value);
        NamedObject(const std::string& name, const T& value);
        ...
    private:
        std::string nameValue;
        T objectValue;
};
```
- NamedObject 声明了含参数的构造函数，则编译器不会生成无参数的默认构造函数；
- NamedObject 既没有声明拷贝构造函数，也没有声明拷贝赋值操作符，因此编译器将按需生成它们。

继续看下述用例：
```c++
NamedObject<int> no1("Smallest Prime Number", 2);
NamedObject<int> no2(no1);                      // 调用拷贝构造函数
```

该编译器生成的拷贝构造函数分别用 `no1.nameValue` 和 `no1.objectValue` 的值来初始化 `no2.nameValue` 和 `no2.objectValue`。其中，
- `nameValue` 的类型是 `string`，标准的 `string` 类型含有拷贝构造函数，因此 `no2.nameValue` 的初始化需要调用以 `no1.nameValue` 的值作为实参的 `string` 拷贝构造函数来完成；
- `objectValue` 的类型是 `int`（模板 `T` 实例化为 `int`），因此 `no2.objectValue` 的初始化需要直接拷贝 `no1.objectValue` 的比特。

上述例子，生成的拷贝赋值操作符也是类似方式。

一般情况下，仅在代码合法并且能证明其有意义时，编译器生成的拷贝赋值操作符和拷贝构造函数表现的行为是相同的。要不然，编译器将拒绝生成拷贝赋值操作符。

考虑下述示例：

```c++
template<typename T>
class NamedObject {
    public:
        // name 的类型不再是 const char *
        NamedObject(std::string& name, const T& value);
        ...
    private:
        std::string& nameValue;     // 现在改成引用
        const T objectValue;        // 现在改成常量
};
```

那么，下述代码会发生什么事情呢？
```c++
std::string newDog("Persephone");
std::string oldDog("Satch");
NamedObject<int> p(newDog, 2);
NamedObject<int> s(oldDog, 36);

p = s;                      // p 的成员变量要怎么赋值？
```

赋值操作应该怎么作用于 `p.nameValue` 呢？
- 如果 `p.nameValue` 的引用变成 `s.nameValue`，即引用本身被修改。然而，C++ 不允许这样操作；
- 如果 `p.nameValue` 引用的 `string` 对象被修改，因此赋值操作需要作用到“持有指向该 `string` 的指针或其引用”的其它对象，即并未直接牵涉到赋值语句中的对象。编译器不能确定该如何做，面对此情形则拒绝编译该赋值行为。

如果你非要含有引用成员的类中支持拷贝赋值操作，你必须自己定义拷贝赋值操作符。

对于含有 `const` 成员的类，也一样。因为修改 `const` 成员的值是非法的，编译器不能确定如何生成赋值函数。

如果基类声明拷贝赋值操作符为 `private`，则编译器对于其继承类不再生成拷贝赋值操作符。毕竟，编译器为继承类生成的拷贝赋值操作符也应该能处理基类的内容（参见12），如果能这样做，而继承类又无权调用该赋值函数，即使编译器要生成它也毫无用武之地。

# 牢记
+ 编译器可以隐式地为类创建默认构造函数、析构函数、拷贝构造函数，以及拷贝赋值构造操作符。