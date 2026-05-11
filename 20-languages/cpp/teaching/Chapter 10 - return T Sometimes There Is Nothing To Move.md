# Chapter 10 - return T Sometimes There Is Nothing To Move

Related:

- [[20-languages/cpp/Learning Path - C++ Object and Resource Semantics]]
- [[20-languages/cpp/teaching/Chapter 9 - Value Category Is Not Lifetime]]
- [[20-languages/cpp/object-resource-semantics/Concept - Copy Elision and C++17 prvalue]]
- [[20-languages/cpp/object-resource-semantics/Concept - RVO and NRVO]]
- [[20-languages/cpp/deep-dives/Deep Dive - RVO Return Slot and Naive Copy Model]]

## Goal

現在我們可以回到一開始的問題：

```cpp
T make() {
    return T{};
}

T y = make();
```

這段 code 很容易被想成：

```text
function 裡先建立一個 temporary T。
return 時把 temporary move 到 caller。
caller 再得到 y。
```

但這個模型不適合 modern C++。

這章要建立一個更好的句子：

```text
Sometimes there is nothing to move.
```

也就是：

```text
有些 return-by-value path 不是先有 source object，
再 copy / move 到 caller。

而是 result object 直接被初始化。
```

這就是 RVO / copy elision 這條線真正重要的地方。

它不是單純的 compiler trick。

它改變的是你理解 object delivery 的方式。

## Cold Open

先看最簡單的版本：

```cpp
T make() {
    return T{};
}
```

先把語法本身講清楚。

這裡的 `T` 不是某個特定 class name。

它是教學裡常用的 placeholder，意思是：

```text
some type T
```

例如你可以把它代換成：

```cpp
std::string{}
Widget{}
Buffer{}
```

`T{}` 則是 C++ 的 brace initialization 寫法。用很白話的方式說，它表示：

```text
用空的 initializer list 建立 / 初始化一個 T。
```

所以：

```cpp
std::string{}
```

代表：

```text
用 no arguments 初始化一個 std::string，
得到一個 empty string。
```

如果是：

```cpp
Widget{}
```

意思大致是：

```text
用 no arguments 初始化一個 Widget。
```

前提是 `Widget` 真的能這樣初始化。

如果某個 type 沒有 default constructor，或不允許 empty-brace initialization，那：

```cpp
T{}
```

本身就是 invalid code。

例如：

```cpp
struct Buffer {
    explicit Buffer(std::size_t n);
};

Buffer{};      // invalid: no default constructor
Buffer{1024};  // ok: calls Buffer(std::size_t)
```

所以這章用 `T{}` 時，先假設：

```text
T 是一個可以用 empty braces 初始化的 type。
```

接下來要問的不是：

```text
T{} 能不能寫？
```

而是：

```text
當 T{} 出現在 return statement 裡時，
C++ 需要先建立一個 temporary T 再 move 嗎？
```

問題是：

```text
這裡會先建立一個 temporary T，
再 move 到 function 的 return value 嗎？
```

如果你剛學完 move，很容易想成：

```text
T{} 是 temporary。
temporary 快死了。
所以 return 時應該 move。
```

這個想法看起來合理。

但它把 `return T{}` 想成了：

```text
source object already exists
-> transfer to destination
```

而這正是需要修正的地方。

## The Naive Return Model

很多人腦中的 return-by-value 模型大概像這樣：

```text
callee:
    create temporary T

return:
    move temporary into return object

caller:
    move return object into y
```

如果用圖表示：

```text
temporary T in callee
    |
    | move
    v
return object
    |
    | move
    v
y in caller
```

這個模型的問題是：

```text
它預設一定有一個獨立 source object。
```

但對：

```cpp
return T{};
```

這個預設不一定成立。

## The Better Model

在 modern C++ 裡，當 function 回傳 `T`，而 return expression 是同型別的 prvalue：

```cpp
T make() {
    return T{};
}
```

可以把它理解成：

```text
T{} 直接初始化 function 的 result object。
```

也就是：

```text
沒有一個獨立 temporary T 先完整出生，
再被 move 到 result object。
```

更好的圖是：

```text
caller needs a T result
        |
        v
construct T directly as the result object
```

所以這裡的重點不是：

```text
compiler 很聰明，把 move 省掉了。
```

而是：

```text
語言規則允許這個 T 直接成為 result object。
```

這就是為什麼說：

```text
Sometimes there is nothing to move.
```

## Why This Is Not Move

前面說過：

```text
move means transfer.
```

Move 的前提是：

```text
source object 已經存在。
destination 要接手 source 背後的 resource / state。
source 留在 valid moved-from state。
```

例如：

```cpp
Buffer a(1024);
Buffer b = std::move(a);
```

這裡有一個很明確的 source object：

```text
a
```

所以 move constructor 可以做：

```text
b.ptr = a.ptr
a.ptr = nullptr
```

但：

```cpp
T make() {
    return T{};
}
```

這裡沒有一個有名字的 local object 要被放棄。

你不是在說：

```text
把 A 已經擁有的 resource 轉交給 B。
```

你是在說：

```text
我要產生 function 的 result。
```

而 `T{}` 可以直接用來初始化那個 result。

所以它不是：

```text
move from temporary
```

而是更接近：

```text
construct the result directly
```

## Connect Back To prvalue

上一章說：

```cpp
T{}
```

是 prvalue expression。

但也提醒過，不要太早把它想成：

```text
先建立 temporary object，
再 copy / move 到目標。
```

現在可以補上原因。

在：

```cpp
T y = T{};
```

或：

```cpp
T make() {
    return T{};
}
```

這類 same-type initialization 裡，`T{}` 不是一定要先 materialize 成一個獨立 temporary，然後再搬到 `y` 或 return object。

它可以直接初始化目標 object。

所以：

```text
prvalue 不應該被初學者固定想成「一個已經存在的 temporary object」。
```

比較好的初始模型是：

```text
prvalue can be used to initialize an object.
```

至於什麼時候需要 temporary materialization，那是更後面的標準細節。

這章先不展開。

## A Useful Test Case

這個例子可以幫你感受差異：

```cpp
struct Token {
    Token() = default;

    Token(const Token&) = delete;
    Token(Token&&) = delete;
};

Token make() {
    return Token{};
}
```

如果你的模型是：

```text
return Token{} 必須 move temporary
```

那這段應該不能成立，因為 move constructor 被 delete 了。

但在 C++17 之後，這種 same-type prvalue return 不需要 copy / move constructor 來完成 return。

`Token{}` 可以直接初始化 return object。

所以這個例子真正說明的是：

```text
return T{} 不應該被理解成「先 temporary，再 move」。
```

這不是說所有 return 都不用 copy / move。

這只是在說：

```text
same-type prvalue return 是一個特別重要的 no-transfer case。
```

## Where The Object Is

你可以用一個比較實作感的 mental model：

```text
caller 需要一個 T result。
callee 知道要 return T{}。
所以 T 可以直接在 result storage 被 constructed。
```

但這裡要小心。

這只是 mental model。

不要把它過度翻譯成固定的 stack layout 或 ABI 規則。

標準層級真正重要的是：

```text
source-level semantics 不要求有一個獨立 temporary 再 move。
```

不同 ABI 可以用 return slot、register、hidden pointer 或其他策略實作。

那些是 systems / ABI layer 的問題。

這章只要先建立 source-level understanding：

```text
return T{} can initialize the result object directly.
```

## What This Means For RVO

這也是為什麼 RVO 不應該被講成：

```text
compiler 幫你把多餘 copy 最佳化掉。
```

那個講法有一部分歷史原因，也有時候可以當 implementation intuition。

但在這條教學主線裡，更重要的是：

```text
RVO / copy elision shows that object delivery is not always transfer.
```

前面我們已經看過：

```text
copy:
    duplicate a usable T

move:
    transfer from an existing source object
```

現在補上第三種情況：

```text
direct construction:
    initialize the destination/result object directly
```

但注意，這個三分法現在才出現是合理的。

因為前面已經先講過：

- copy 為什麼不是 bytes copy；
- move 為什麼不是 faster copy；
- `std::move` 為什麼不 move；
- value category 為什麼不是 lifetime。

所以現在說：

```text
return T{} 有時候根本沒有東西要 move。
```

才不會變成空泛口號。

## A Small Boundary

這章目前只談這種形狀：

```cpp
T make() {
    return T{};
}
```

也就是：

```text
function returns T
return expression is a same-type prvalue
```

不要把這章的結論直接套到所有 return：

```cpp
T make() {
    T x;
    return x;
}
```

或：

```cpp
T make() {
    T x;
    return std::move(x);
}
```

這兩個 case 有 named local object。

它們牽涉 NRVO、implicit move fallback、以及 `std::move` 如何改變 return expression。

那是下一章的主題。

## Why This Matters For The Main Thesis

這章把整條路線往前推了一步。

我們一開始的疑問是：

```text
return by value 會不會很貴？
```

後來發現更深的問題是：

```text
caller 那邊到底怎麼得到一個 T？
```

現在答案開始清楚了：

```text
有時候是 copy。
有時候是 move。
有時候根本不是 transfer，而是直接初始化 result object。
```

所以 C++ 的重點不是：

```text
到處 std::move。
```

也不是：

```text
相信 compiler optimization。
```

更精準地說是：

```text
用 type operations 和 language rules，
讓 object 在正確的位置、用正確的語意被建立。
```

## What This Chapter Does Not Explain Yet

這章還不處理：

```text
return x
return std::move(x)
NRVO
implicit move from local variables
C++23 return move rule changes
ABI return slot details
temporary materialization conversion
```

這些都重要，但如果現在全部塞進來，會破壞這章的焦點。

這章只要把這件事講清楚：

```text
return T{} is not best understood as temporary-then-move.
```

## Three Things To Take Away

1. `return T{}` 這種 same-type prvalue return，在 modern C++ 裡可以直接初始化 result object，不需要先建立獨立 temporary 再 move。
2. Move 的前提是有一個 existing source object 可以被放棄；`return T{}` 的重點是 result object 可以直接被 constructed。
3. RVO / copy elision 的教學價值不是「compiler trick」，而是展示 object delivery 有時候根本不是 transfer。

## Next Question

```text
那 named local return 呢？

T make() {
    T x;
    return x;
}

T make_move() {
    T x;
    return std::move(x);
}

這兩種差在哪？
```

Next:

- [[20-languages/cpp/teaching/Chapter 11 - return x vs return std move x]]
