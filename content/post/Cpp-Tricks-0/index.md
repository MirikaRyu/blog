+++
date = '2026-04-13T16:21:57+08:00'
title = 'C++ 中的奇技淫巧・零'

tags = ['C++', 'Template', 'Macro']
categories = ['C++']
image = 'cover.webp'

draft = true
+++

众所周知，在 C++ 中有各种各样奇奇怪怪的 tricks。下面就收集了一些个人感觉比较有趣的特性或者写法。

## 使用纯类型系统判断宏是否定义
本质上就是要找到一种语言构造，使得一个字面量或者一个未声明的标识符都能够嵌入该构造中，并产生不同的结果。

首先不难发现存在以下相似的构造：
```cpp
int(__MACRO__);
int(1);
```
上面的是一个声明语句，声明了一个 `__MACRO__` 的整型变量；下面的是显式类型转换，将 `1` 转换到整型然后丢弃。

但一个单独的声明语句不太好发挥作用，因此还需要继续放入其它的结构中，同时还要和转型表达式兼容 🤔。

然后就有神人想到了这个：
```cpp
int(int(__MACRO__))
```
当 `__MACRO__` 未定义的时候，最外层为 `int()` 是一个返回值为 `int` 的函数，而内层的 `int(__MACRO__)` 则声明了一个整型的函数参数，
变量名为 `__MACRO__`，因此最后的结果是一个函数类型。当 `__MACRO__` 被替换成数字之后，整个就变为了一个简单的转型表达式，
结果为一个值。

不过需要注意上面这个必须要在能够同时接受类型和值的上下文中出现才能够实现不同的解析方式，即模板形参、`sizeof`、`alignof`
这一类，但其中只有模板上下文是可以被我们控制的。

最后再根据类型进行分发即可：
```cpp
template <typename>
consteval bool detect(void) { return false; }

template <auto>
consteval bool detect(void) { return true; }

static_assert(detect<int(int(__MACRO__))>(), "MACRO is not defined");
```
