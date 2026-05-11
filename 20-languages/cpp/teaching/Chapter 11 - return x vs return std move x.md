# Chapter 11 - return x vs return std move x

Related:

- [[20-languages/cpp/Learning Path - C++ Object and Resource Semantics]]
- [[20-languages/cpp/teaching/Chapter 8 - std move Does Not Move]]
- [[20-languages/cpp/teaching/Chapter 9 - Value Category Is Not Lifetime]]
- [[20-languages/cpp/teaching/Chapter 10 - return T Sometimes There Is Nothing To Move]]
- [[20-languages/cpp/object-resource-semantics/Concept - RVO and NRVO]]
- [[20-languages/cpp/object-resource-semantics/Concept - std move vs Move Constructor]]
- [[20-languages/cpp/deep-dives/Deep Dive - std move xvalue and Return Path Selection]]

## Goal

上一章處理了：

```cpp
T make() {
    return T{};
}
```

這種 same-type prvalue return 的重點是：

```text
sometimes there is nothing to move.
```

現在回到另一個更容易寫到的形式：

```cpp
T make() {
    T x;
    return x;
}
```

這裡真的有一個 named local object：

```text
x
```

所以問題變成：

```text
既然 x 是 local object，
它快要離開 scope，
那我是不是應該幫 compiler 寫 std::move(x)？
```

這章要拆掉這個直覺：

```text
return std::move(x) is usually not the safer default.
```

## Cold Open

先比較兩段 code：

```cpp
#include <utility>

T make() {
    T x;
    return x;
}

T make_move() {
    T x;
    return std::move(x);
}
```

問題是：

```text
哪一個比較好？
```

很多人的直覺會是：

```text
copy 很貴。
move 通常比較便宜。
所以 return std::move(x) 比較保險。
```

這個直覺很自然。

但它漏掉了 C++ return statement 的另一條路：

```text
有時候根本不需要 transfer。
```

## The Naive Model

如果只用 copy / move 的角度看，可能會想成：

```text
return x:
    x 是 lvalue
    所以 copy

return std::move(x):
    std::move(x) 是 xvalue
    所以 move
```

於是結論就會變成：

```text
return std::move(x) 比 return x 好。
```

但這個模型不完整。

它把 return-by-value 想成只有兩種可能：

```text
copy
move
```

可是前一章剛建立過第三種情況：

```text
direct construction / copy elision
```

也就是：

```text
不是從 source object transfer，
而是直接讓 result object 形成。
```

## What `return x` Allows

先看：

```cpp
T make() {
    T x;
    return x;
}
```

這裡的 `x` 是：

```text
local automatic object
same type as function return type
returned by name
```

這個形狀很重要。

它讓 `x` 成為 NRVO candidate。

NRVO 是 Named Return Value Optimization：

```text
Named local object 可以直接被建構成 function result object。
```

也就是概念上可以變成：

```text
caller needs T result
        |
        v
construct local x directly as the result object
```

這時候不是：

```text
copy x to result
```

也不是：

```text
move x to result
```

而是：

```text
x 就是那個 result object。
```

所以 `return x` 保留了一個很好的可能性：

```text
no copy
no move
direct construction of the result
```

## What If NRVO Does Not Happen?

Named RVO 和上一章的 `return T{}` 不完全一樣。

`return T{}` 這種 same-type prvalue return，在 C++17 之後是 guaranteed copy elision 的核心案例。

但：

```cpp
T make() {
    T x;
    return x;
}
```

這種 named local return 的 NRVO 不是同一種 guarantee。

所以你還是要問：

```text
如果 compiler 沒有做 NRVO，會怎樣？
```

這時候 C++ return statement 還有特殊規則。

雖然在一般 expression 裡：

```cpp
x
```

是 lvalue expression，但在 return local object 的情境下，C++ 可以嘗試用 move path 來初始化 result。

所以實務上可以把 `return x` 的 fallback 想成：

```text
try NRVO first;
if not elided, move if possible;
otherwise copy if possible.
```

這不是完整標準 wording。

但作為第一輪 mental model，它抓到了重點：

```text
return x 不是「一定 copy」。
```

## What `return std::move(x)` Changes

現在看：

```cpp
T make_move() {
    T x;
    return std::move(x);
}
```

上一章已經說過：

```text
std::move(x) does not move.
```

它做的是：

```text
把 expression 轉成 xvalue。
```

所以在 return statement 裡：

```cpp
return std::move(x);
```

意思不是：

```text
幫 compiler 做最佳化。
```

而是：

```text
return expression 不再是 plain local variable name `x`。
return expression 變成 xvalue expression `std::move(x)`。
```

這會影響 return path。

最重要的後果是：

```text
它不再是 NRVO 的典型 candidate shape。
```

`return x` 的形狀是：

```cpp
return x;
```

`return std::move(x)` 的形狀是：

```cpp
return some_expression_involving_x;
```

這兩個對 NRVO 不是同一回事。

## The Tradeoff

可以把兩者先整理成：

```text
return x:
    keeps NRVO candidate shape
    if NRVO happens: no copy, no move
    if NRVO does not happen: may move/copy as fallback

return std::move(x):
    produces an xvalue expression
    can select move constructor
    but usually prevents the NRVO path
```

所以 `return std::move(x)` 不是更安全。

它比較像是：

```text
放棄「可能直接建構 result object」，
改成明確要求從 xvalue 初始化 result。
```

如果 move constructor 很便宜，這可能看起來也還好。

但從 C++ object delivery 的角度看：

```text
no transfer is better than cheap transfer.
```

如果 result object 可以直接形成，通常沒有理由先強迫它走 move path。

## Why The Compiler Does Not Need Your Help Here

這裡有一個心理模型要修正。

你寫：

```cpp
return std::move(x);
```

通常是因為你想表達：

```text
x 快死了。
請不要 copy。
請 move。
```

但 compiler 對這個情境本來就有特殊規則。

它知道：

```text
x 是 local object。
return 後 x 的 lifetime 要結束。
```

所以 `return x` 已經不是普通的：

```cpp
T y = x;
```

你不需要用 `std::move` 來提醒 compiler：

```text
這個 local 快死了。
```

更重要的是，你的提醒可能把更好的選項關掉：

```text
NRVO / no transfer
```

所以常見 guideline 是：

```text
return local variable by name.
do not write return std::move(local).
```

這也是為什麼一些 compiler 會對這種 code 給 warning，例如：

```text
pessimizing move
```

意思大概是：

```text
你以為你在幫忙 move，
但你可能讓 copy elision / NRVO 變差。
```

## A Trace You Should Not Over-Trust

很多人會想用 constructor logging 驗證：

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

    T(T&&) noexcept {
        std::cout << "move\n";
    }
};

T make() {
    T x;
    return x;
}

T make_move() {
    T x;
    return std::move(x);
}
```

這種實驗可以幫助觀察，但不要把輸出當成唯一真理。

原因是：

```text
copy elision / NRVO 會改變你看到的 constructor calls。
不同 compiler、optimization level、standard mode 可能讓 trace 看起來不同。
```

這裡更穩的理解方式是 source-level rule shape：

```text
return x:
    preserves NRVO candidate shape

return std::move(x):
    changes the expression to xvalue
    removes the plain-name shape needed for NRVO
```

如果你真的要做實驗，可以觀察不同情況。

但教學主線不要讓讀者以為：

```text
看 log 出現 move / copy 才是語意本身。
```

Constructor log 只是觀察工具。

不是 object model。

## The Important Distinction

現在可以把 Chapter 10 和 Chapter 11 接在一起。

`return T{}`：

```text
same-type prvalue return
result object can be initialized directly
usually think: there is no source object to move
```

`return x`：

```text
named local object return
keeps NRVO candidate shape
if NRVO happens, x is the result object
if not, move/copy fallback may happen
```

`return std::move(x)`：

```text
explicit xvalue return expression
can call move constructor
but gives up the normal NRVO candidate shape
```

這三個不是同一件事。

所以不要把所有 return-by-value 都想成：

```text
先有 temporary / local，
再 move 到 caller。
```

更好的模型是：

```text
return by value asks:
    how is the result object initialized?
```

答案可能是：

```text
direct initialization
NRVO
move fallback
copy fallback
```

而不是一律：

```text
copy or move from callee to caller
```

## When `return std::move(x)` Might Be Intentional

這章不是說：

```text
return std::move(x) 永遠不可能出現。
```

而是說：

```text
不要把它當成 return local object 的 default habit。
```

如果 return expression 本來就不是 NRVO candidate，例如：

```cpp
T make(bool cond) {
    T a;
    T b;

    if (cond) {
        return a;
    }

    return b;
}
```

這裡每個 branch 仍然是 local variable return，可能有 move fallback。

但如果你寫的是更複雜的 expression，或你真的要從某個 member / container / parameter 轉出 ownership，那就要根據實際語意判斷。

例如：

```cpp
T take_from(MemberOwner& owner) {
    return std::move(owner.value);
}
```

這不是 local NRVO case。

這裡你是在明確表達：

```text
我要從 owner.value 轉出 resource。
```

所以規則不是：

```text
never use std::move in return.
```

而是：

```text
when returning a local object by value,
prefer return x;

use std::move only when you really mean ownership transfer
from an existing object and you are not giving up a better no-transfer path.
```

## What This Chapter Does Not Explain Yet

這章刻意不展開：

```text
C++23 move-eligible expression rule changes
exact overload-resolution wording for implicit move
throw expressions and coroutine parameter copies
ABI return slot details
all cases where NRVO is disallowed
```

這些都可以放到 deeper reference note。

這章只要讓讀者建立正確習慣：

```text
return x is not automatically worse than return std::move(x).
```

甚至在常見 local-return case 裡：

```text
return x is usually the better expression.
```

## Three Things To Take Away

1. `return x` 保留 named local object 的 NRVO candidate shape；如果 NRVO 發生，就沒有 copy 也沒有 move。
2. `return std::move(x)` 把 return expression 變成 xvalue，通常可以選 move constructor，但也通常放棄 NRVO 這條 no-transfer path。
3. Return by value 的核心問題不是「copy 還是 move」，而是 result object 如何被初始化；有時候最好的答案是 no transfer。

## Next Question

```text
現在我們看過 return by value、copy、Buffer、move、std::move、value category、RVO / NRVO。

這些真的只是 C++ 零散規則嗎？
還是它們其實都在解同一個更大的問題？
```

Next:

- [[20-languages/cpp/teaching/Chapter 12 - From C Convention To Cpp Semantic Lifting]]
