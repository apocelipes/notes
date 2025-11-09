猜猜下面这段代码的输出是什么：

```c++
template <typename T>
struct Base {
    void DoThings() {
        std::cout << "A\n";
    }
};

template <typename T>
struct Derived: Base<T> {
    void Do() {
        DoThings();
    }
};

int main() {
    Derived<int> d;
    d.Do();
}
```

肯定有人会说是A，但实际上是编译错误：

```console
test.cpp: In member function 'void Derived<T>::Do()':
test.cpp:9:17: error: there are no arguments to 'DoThings' that depend on a template parameter, so a declaration of 'DoThings' must be available [-Wtemplate-body]
    9 |                 DoThings();
      |                 ^~~~~~~~
test.cpp:9:17: note: (if you use '-fpermissive', G++ will accept your code, but allowing the use of an undeclared name is deprecated)
```

给的报错信息很让人迷惑，因为`DoThings`是明确声明定义在`Base<T>`中的，这里居然在说它未被声明。

这其实是c++的Two Phase Lookup导致的。

`Two Phase Lookup`如其字面意思，对于任何模板代码，编译器需要进行两次检查：

1. Phase 1，第一步检查，只检查模板代码是否有语法错误，但涉及到和模板类型参数相关的部分会跳过。检查的范围包括是否有明显的语法错误比如用了不存在的关键字、少了分号等，其中也会检查哪些和模板类型参数无关的函数、类型、方法是否已经被声明，这和编译器检查普通代码的流程很相似
2. Phase 2，这一步会往模板的参数里带入实际的类型，编译器会重新推导整个模板代码在当前的类型下是否合法

两步骤是为了更快速地将类型参数不相关的问题排除，这样在保证模板代码语法正确性的同时尽量保证了泛型代码的灵活性，理想中也能让模板的编写者更快发现问题而不是把问题延迟到类型推导之后。

但坏处就是会让模板产生一下诡异的编译错误了，比如上面的`DoThings`。`DoThings`在这里是非限定名称，但没有参数，同时它也和`Derived`模板的类型参数不直接相关，这导致对`DoThings`的检查会在Phase 1执行，而Phase 1会忽略所有的模板参数相关内容，这导致`Base<T>`在这时不可见，而我们又没有在其他地方定义`DoThings`，所以编译器认为我们在使用一个未声明的符号，于是报了语法错误。

解决方法也很简单，让`DoThings`和类型参数相关即可，或者通过this去调用，this代表了泛型模板类自身，也算和模板参数相关：

```diff
template <typename T>
struct Derived: Base<T> {
    void Do() {
-       DoThings();
+       this->DoThings();
+       // Base<T>::DoThings(); 也可以
    }
};
```

另外如果我们提供了自由函数`DoThings`，那么在Phase 1中就会把对应的名字认定为是在调用自由函数，这时编译器不再报错，但`Base<T>`的方法永远调用不到了：

```c++
template <typename T>
struct Base {
    void DoThings() {
        std::cout << "A\n";
    }
};

void DoThings() {
    std::cout << "B\n";
}

template <typename T>
struct Derived: Base<T> {
    void Do() {
        DoThings();
    }
};

int main() {
    Derived<int> d;
    d.Do(); // 输出B，自由函数DoThings被调用
}
```

这很违反直觉，因为普通的非模板子类在这种时候会去基类的作用域里寻找同名的方法，但因为`Two Phase Lookup`，编译器在Phase 1把函数绑定到了全局的自由函数上，这导致了非预期的结果。

## 总结

模板会有`Two Phase Lookup`做两遍检查，因此它和普通的代码行为上会有区别。

除了上面说的让方法和模板参数关联，其他补救措施还有很多，一种在GCC给出的编译报错里：加上`-fpermissive`启用permissive模式。在这个模式下不会进行`Two Phase Lookup`，模板会在实例化的时候再做检查，可以避免报错，但实测gcc-15上无法避免错误调用自由函数的问题。另外permissive模式会大幅改变语言和编译器的行为，贸然启用会出现很多意外问题。因此这一措施我并不推荐。

让方法名和模板参数相关也不能解决所有问题，因为还有很多时候我们需要利用非限定名称来自动选取合适的函数/方法，碰到这种情况就只能特殊场景特殊处理了。

简单地说，没有银弹，不存在一种万金油方法彻底规避这类错误。这也只是c++模板黑暗面的冰山一角罢了。
