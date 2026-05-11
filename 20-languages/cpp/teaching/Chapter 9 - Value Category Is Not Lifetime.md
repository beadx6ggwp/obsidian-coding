# Chapter 9 - Value Category Is Not Lifetime

Related:

- [[20-languages/cpp/Learning Path - C++ Object and Resource Semantics]]
- [[20-languages/cpp/teaching/Chapter 8 - std move Does Not Move]]
- [[20-languages/cpp/object-resource-semantics/Concept - Value Categories - lvalue xvalue prvalue]]
- [[20-languages/cpp/deep-dives/Deep Dive - Value Categories Beyond std move]]
- [Arthur O'Dwyer - Value category is not lifetime](https://quuxplusone.github.io/blog/2019/03/11/value-category-is-not-lifetime/)

## Goal

上一章說：

```text
std::move(x) does not move.
```

它只是把 expression 轉成比較容易選到 move operation 的形式。

這章要補上那個缺口：

```text
expression category 到底在描述什麼？
```

但這章不打算變成完整標準分類表。

這章只要讓你抓住一句話：

```text
value category describes expressions.
lifetime describes objects.
```

這兩件事相關，但不是同一件事。

## The Three Expressions We Care About

先看三個 expression：

```cpp
T x;

x;
std::move(x);
T{};
```

它們看起來都和 `T` 有關。

但它們不是同一種 expression。

先用最少的 vocabulary：

```text
x:
    lvalue expression

std::move(x):
    xvalue expression

T{}:
    prvalue expression
```

現在先不要急著展開完整 taxonomy：

```text
glvalue
rvalue
lvalue
xvalue
prvalue
```

那張表以後可以查。

這裡真正重要的是：

```text
value category 是 expression 的性質，
不是 object 本身的壽命。
```

## Expression vs Object

先分清楚兩件事：

```text
object:
    程式執行時存在的東西。
    有 storage。
    有 lifetime。

expression:
    source code 裡的一段語法。
    有 type。
    有 value category。
```

例如：

```cpp
T x;
```

這行建立了一個 object `x`。

這個 object 的 lifetime 大概從宣告那行開始，到 scope 結束。

但是當你在 code 裡寫：

```cpp
x
```

這個 `x` 是一個 expression。

它的 type 是：

```text
T
```

它的 value category 是：

```text
lvalue
```

所以：

```text
object x:
    has lifetime

expression x:
    has value category
```

不要把這兩層混在一起。

## `x` Is An lvalue Expression

看這段：

```cpp
T x;
```

只要你寫：

```cpp
x
```

這個 expression 是 lvalue。

即使 `x` 快要離開 scope，expression `x` 仍然是 lvalue。

例如：

```cpp
T make() {
    T x;
    return x;
}
```

在 `return x;` 裡，`x` 仍然是 lvalue expression。

不要因為：

```text
function 快結束了，
x 快死了
```

就以為 `x` 這個 expression 自動變成 rvalue。

這就是：

```text
value category is not lifetime.
```

`x` 快不快死，是 object lifetime / return rule 的問題。

`x` 這個 expression 是不是 lvalue，是 expression category 的問題。

## `std::move(x)` Is An xvalue Expression

上一章說過：

```cpp
std::move(x)
```

大致上像：

```cpp
static_cast<T&&>(x)
```

它沒有 move object。

它做的是：

```text
把 expression `x`
轉成 xvalue expression。
```

重要的是：

```text
std::move(x) still refers to the same object x.
```

它沒有建立新 object。

它沒有結束 `x` 的 lifetime。

它只是讓後續 operation 可以把 `x` 當成一個可以被接手 resource 的 source。

所以：

```cpp
T x;
std::move(x);
```

不會讓 `x` 死掉。

也不會讓 `x` 自動 moved-from。

但：

```cpp
T y = std::move(x);
```

可能選到 move constructor，然後 move constructor 才真的 transfer resource。

## `T{}` Is A prvalue Expression

再看：

```cpp
T{}
```

這是一個 prvalue expression。

暫時可以把它理解成：

```text
這是一個直接產生 T 值的 expression，
可以用來初始化一個 T object。
```

例如：

```cpp
T y = T{};
```

這裡 `T{}` 可以初始化 `y`。

但要小心，不要太早把 `T{}` 想成：

```text
先建立一個 temporary object，
再 copy / move 到 y。
```

這就是後面 return-by-value / C++17 prvalue 會回來處理的地方。

現在只要先知道：

```text
T{} 的 value category 是 prvalue。
```

它和：

```cpp
x
std::move(x)
```

不同。

## Why Value Category Matters

Value category 會影響 overload resolution。

例如：

```cpp
void use(const T&);
void use(T&&);
```

現在：

```cpp
T x;

use(x);
use(std::move(x));
use(T{});
```

大致上可以這樣想：

```text
use(x):
    x 是 lvalue expression
    選 const T& overload

use(std::move(x)):
    std::move(x) 是 xvalue expression
    可以選 T&& overload

use(T{}):
    T{} 是 prvalue expression
    可以選 T&& overload
```

這就是為什麼 `std::move` 有用。

它不是因為自己 move。

而是因為它改變 expression category，讓後續 overload resolution 可以選到 move path。

## Named Rvalue References Are Still lvalues

這裡有一個常見陷阱。

假設你寫：

```cpp
void f(T&& t) {
    use(t);
}
```

很多人會以為：

```text
t 的 type 是 T&&，
所以 t 是 rvalue。
```

但在 expression 裡：

```cpp
t
```

是 lvalue expression。

因為 `t` 有名字。

所以：

```cpp
void f(T&& t) {
    use(t);            // t is lvalue expression
    use(std::move(t)); // xvalue expression
}
```

這再次說明：

```text
type and value category are different axes.
```

`t` 的 declared type 可以是 `T&&`。

但 expression `t` 的 value category 仍然是 lvalue。

## Lifetime Is A Different Question

現在回到這章標題：

```text
Value Category Is Not Lifetime
```

看三個例子：

```cpp
T x;

x;
std::move(x);
T{};
```

你不能用 value category 直接推論 object 活多久。

例如：

```cpp
std::move(x)
```

是 xvalue expression。

但它 refer to 的 object `x` 仍然活到 scope 結束。

所以 xvalue 不等於：

```text
object 馬上死掉
```

也不等於：

```text
object 已經被 move
```

它比較像是在說：

```text
這個 expression 可以被後續 operation 當成快要被放棄的來源。
```

但真正發生什麼，還要看後面有沒有 move constructor / move assignment / binding / initialization。

## How This Prepares Return By Value

現在我們終於有足夠 vocabulary 回到一開始的問題。

```cpp
T make() {
    T x;
    return x;
}
```

在這裡：

```text
x 是 lvalue expression。
```

但這不代表：

```text
一定會 copy。
```

也不代表你應該馬上寫：

```cpp
return std::move(x);
```

因為 return statement 有自己的規則，包含 NRVO / implicit move fallback 等等。

再看：

```cpp
T make() {
    return T{};
}
```

這裡：

```text
T{} 是 prvalue expression。
```

這也不能用「先 temporary 再 move」的舊模型想。

下一段回到 return by value 時，會用這些 vocabulary 區分：

```text
return T{}
return x
return std::move(x)
```

這三個 case 到底差在哪裡。

## What This Chapter Does Not Explain Yet

這章仍然沒有完整展開：

```text
glvalue
rvalue
C++17 prvalue materialization
temporary materialization conversion
NRVO
implicit move
```

那些都可以作為延伸或在 return-by-value 章節補上。

這章只要建立：

```text
expression has type and value category.
object has lifetime.
不要把兩者混成同一件事。
```

## Three Things To Take Away

1. `x` 是 lvalue expression，即使 `x` 快要離開 scope。
2. `std::move(x)` 是 xvalue expression，但它不會改變 `x` 的 lifetime，也不會自己 transfer resource。
3. `T{}` 是 prvalue expression；它要在 return-by-value 章節重新理解，不能先假設一定有 temporary 再 move。

## Next Question

```text
現在我們知道 copy、move、std::move、value category 了。
那回到一開始的 return by value：
哪些情況是 copy？
哪些情況是 move？
哪些情況是連 move 都不需要？
```

Next:

- [[20-languages/cpp/teaching/Chapter 10 - return T Sometimes There Is Nothing To Move]]
