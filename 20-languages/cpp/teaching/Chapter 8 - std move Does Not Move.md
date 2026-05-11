# Chapter 8 - std move Does Not Move

Related:

- [[20-languages/cpp/Learning Path - C++ Object and Resource Semantics]]
- [[20-languages/cpp/teaching/Chapter 7 - Move Is Ownership Transfer Not Faster Copy]]
- [[20-languages/cpp/object-resource-semantics/Concept - std move vs Move Constructor]]
- [[20-languages/cpp/deep-dives/Deep Dive - std move xvalue and Return Path Selection]]

## Goal

前面幾章已經把 move 的意義講清楚：

```text
copy means duplication.
move means transfer.
```

但現在會遇到一個很容易誤解的工具：

```cpp
std::move(x);
```

這章要建立一個非常重要的句子：

```text
std::move does not move.
```

真正 transfer resource 的不是 `std::move`。

真正 transfer resource 的是：

```text
move constructor
move assignment
```

`std::move` 只是讓後面的 operation 有機會選到 move path。

## Cold Open

先看這段 code：

```cpp
#include <utility>

T x;
std::move(x);
```

問題是：

```text
這行執行完，x 被 move 了嗎？
```

很多人的直覺會是：

```text
std::move(x)
-> move x
-> x 變成 moved-from
```

但這個直覺是錯的。

`std::move(x)` 這個 expression 本身沒有搬任何 resource。

它沒有呼叫 `T` 的 move constructor。

它也沒有呼叫 `T` 的 move assignment。

如果你只是單獨寫：

```cpp
std::move(x);
```

那通常什麼 transfer 都不會發生。

## A Small Trace

假設有一個 type：

```cpp
#include <iostream>
#include <utility>

struct T {
    T() {
        std::cout << "default\n";
    }

    T(const T&) {
        std::cout << "copy\n";
    }

    T(T&&) {
        std::cout << "move\n";
    }
};
```

現在寫：

```cpp
int main() {
    T x;
    std::move(x);
}
```

你不應該期待看到：

```text
move
```

因為這裡沒有建立另一個 `T`。

也沒有把 `x` assign 給另一個 `T`。

`std::move(x)` 只是產生一個 expression，然後這個 expression 沒有被拿去做任何事。

如果改成：

```cpp
int main() {
    T x;
    T y = std::move(x);
}
```

這時候才會呼叫 move constructor：

```text
default
move
```

差別在這裡：

```text
std::move(x):
    只改變 expression 的形式。

T y = std::move(x):
    用那個 expression 初始化 y，
    因此可能選到 move constructor。
```

## What std::move Really Is

概念上，`std::move(x)` 大致就是一個 cast：

```cpp
static_cast<T&&>(x)
```

它做的事情不是：

```text
move the object
```

而是：

```text
把 x 這個 expression 轉成一種「可以被當作快要過期的值」的 expression。
```

比較精準地說：

```text
x:
    lvalue expression

std::move(x):
    xvalue expression
```

這章先不用完整展開 value category 表。

先抓住：

```text
x 是有名字的 object，所以 expression `x` 是 lvalue。
std::move(x)` 把這個 expression 變成 xvalue。
```

這個轉換會影響 overload resolution。

也就是：

```cpp
T y = x;            // usually calls copy constructor
T z = std::move(x); // may call move constructor
```

## Permission Signal

可以把 `std::move(x)` 先理解成一個 permission signal：

```text
我允許你把 x 當成可以被移走 resource 的 source。
```

但 permission 不等於 transfer 已經發生。

真正發生 transfer 的地方，是下一個 operation：

```cpp
T y = std::move(x);
```

或：

```cpp
y = std::move(x);
```

第一個可能呼叫：

```text
move constructor
```

第二個可能呼叫：

```text
move assignment
```

所以：

```text
std::move prepares the expression.
move constructor / move assignment performs the transfer.
```

## Why Named Objects Need std::move

這裡有一個看起來反直覺的點。

即使 `x` 的 type 是可以 move 的：

```cpp
Buffer x(1024);
```

expression `x` 仍然是 lvalue。

所以：

```cpp
Buffer y = x;
```

看起來像是：

```text
請從一個 named object 建立 y。
```

這通常會走 copy path。

因為 compiler 不能只因為 `Buffer` 支援 move，就假設你願意放棄 `x`。

如果你真的要表達：

```text
我之後不需要 x 原本擁有的 resource 了。
可以把它交給 y。
```

你才寫：

```cpp
Buffer y = std::move(x);
```

這行的意思不是：

```text
std::move 把 x 搬走。
```

而是：

```text
把 x 這個 expression 標記成可以被 move constructor 接手的 source。
```

## What Happens To x?

如果真的呼叫了 move constructor：

```cpp
Buffer y = std::move(x);
```

那 `x` 會進入一個 moved-from state。

但這裡也要小心。

`x` 沒有被 destroy。

`x` 還活著。

它之後仍然會離開 scope，仍然會跑 destructor。

所以 move constructor 必須讓 `x` 留在一個 safe-to-destroy state。

例如對 `Buffer`：

```text
before:
    x.ptr ----> heap block

after:
    y.ptr ----> heap block
    x.ptr ----> nullptr
```

所以：

```text
std::move(x) alone:
    x 沒有變。

Buffer y = std::move(x):
    move constructor 可能 transfer resource，
    x 變成 moved-from but still alive。
```

這兩件事一定要分開。

## The Common Mistake

常見錯誤是把這三件事混在一起：

```text
std::move
move constructor
moved-from state
```

但正確順序應該是：

```text
std::move(x)
    產生 xvalue expression

T y = std::move(x)
    overload resolution 選到 move constructor

move constructor
    transfer resource
    leave source safe to destroy
```

所以如果你只看到：

```cpp
std::move(x);
```

你不能說：

```text
x 已經被 move 了。
```

你只能說：

```text
這個 expression 被 cast 成可以被 move-from 的形式。
但沒有後續 operation 使用它，所以沒有 transfer。
```

## What This Means For Return

現在先不要深入 `return x` vs `return std::move(x)`。

但可以先留一個伏筆：

```cpp
T make() {
    T x;
    return std::move(x);
}
```

很多人會寫這段，是因為他們以為：

```text
std::move(x) 會幫 compiler move。
```

現在你已經知道：

```text
std::move(x) 本身不 move。
它只是改變 return expression 的 value category。
```

這會影響後面 return path 的選擇。

但那要等 RVO / NRVO 回來時再完整講。

現在只要先把錯誤模型拿掉：

```text
std::move is not an action that moves the object.
```

## What This Chapter Does Not Explain Yet

這章只用了兩個 value category 詞：

```text
lvalue
xvalue
```

下一章才會整理：

```text
value category is about expressions,
not object lifetime.
```

這章也還不完整處理：

```text
prvalue
rvalue
glvalue
return x
return std::move(x)
RVO / NRVO
```

先不要急著把整張表背起來。

先把這句話刻進腦中：

```text
std::move does not move.
```

## Three Things To Take Away

1. `std::move(x)` 本身不呼叫 move constructor，也不 transfer resource。
2. `std::move(x)` 大致上是 `static_cast<T&&>(x)`，讓 expression 變成可以選到 move operation 的形式。
3. 真正 move 的是 move constructor / move assignment；如果它們沒有被呼叫，object 就沒有被 move。

## Next Question

```text
如果 std::move 只是改變 expression，
那 expression category 到底在描述什麼？
```

Next:

- [[20-languages/cpp/teaching/Chapter 9 - Value Category Is Not Lifetime]]
