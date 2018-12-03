# 如何驳回编译器编写你不需要的函数？
```c++
class HomeForSale {...};
```

x
```c++
class HomeForSale {
    public:
        ...
    private:
        ...
        HomeForSale(const HomeForSale&);            // 只有声明，没有实现。
        HomeForSale& operator=(const HomeForSale&);
};
```

# 牢记
+ 为了驳回编译器自动创建你不需要的函数，可以将相应的成员函数声明为 `private` 并且不予实现。也可以参照基类 `Uncopyable` 的做法。