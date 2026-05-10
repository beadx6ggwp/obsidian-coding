# Chapter 2 - Copy Is Not Just Moving Bytes

Related:

- [[20-languages/cpp/Learning Path - C++ Object and Resource Semantics]]
- [[20-languages/cpp/teaching/Chapter 1 - How Many Objects Are In This Code]]
- [[20-languages/cpp/object-resource-semantics/Concept - C++ Type as Operations and Invariants]]

## Goal

上一章留下的問題是：

```text
如果 return by value 真的需要 copy，
那 copy 到底是什麼？
```

這章先不講 `move`。

也先不講完整的 `RVO` / `NRVO`。

這章只要建立一個觀念：

```text
copy 不是單純把 A 的 bytes 搬到 B。

copy 之後，B 必須是一個可以正常使用、正常銷毀的 T。
```

## The Easy Case: Plain Data

先看一個很簡單的 type：

```cpp
struct Point {
    int x;
    int y;
};

Point a{1, 2};
Point b = a;
```

這裡你很容易把 copy 想成：

```text
b.x = a.x
b.y = a.y
```

這個想法在這種 type 上通常不會造成問題。

因為 `Point` 裡面只有兩個 `int`。

copy 之後：

```text
a 和 b 各自有自己的 x / y。
b 可以正常使用。
b 可以正常銷毀。
```

所以在這個例子裡，`copy = 複製 fields` 這個 mental model 看起來成立。

但這只是因為 `Point` 太簡單。

## The Naive Model

很多人會從這種簡單 case 推出一個更大的直覺：

```text
copy = 把 A 的 memory 內容複製一份到 B
```

也就是：

```text
same bytes
-> same object meaning
```

這個推論很危險。

因為 C++ object 不一定只是 fields 的集合。

一個 object 可能管理：

```text
heap memory
file handle
mutex lock
socket
GPU resource
reference-counted control block
```

當一個 object 管理這些東西時，copy 就不只是「讓 bytes 看起來一樣」。

copy 後的 `B` 必須回答更實際的問題：

```text
B 能不能正常使用？
B 要不要釋放某個 resource？
A 和 B 是各自擁有一份，還是共享同一份？
A 死掉時會不會影響 B？
B 死掉時會不會破壞 A？
```

這些不是 raw bytes 自己能回答的問題。

## A More Useful Definition

這章先用比較口語的說法：

```text
copy 之後，
B 應該是一個新的 T object。
B 可以正常使用。
B 可以正常銷毀。
B 代表的內容應該符合這個 type 對 copy 的承諾。
```

這裡故意不急著說 `invariant` 或 `semantic equivalence`。

那些詞後面會出現。

現在只要抓住：

```text
copy 不是 byte operation。
copy 是 T 這個 type 定義出來的 operation。
```

也就是：

```text
Point 的 copy 可以只是複製兩個 int。
std::string 的 copy 必須讓兩個 string 都能正常使用、正常銷毀。
resource owner 的 copy 必須決定 resource ownership 要怎麼處理。
```

## Example: `std::string`

看一個比 `Point` 更接近真實 C++ 的例子：

```cpp
#include <string>

std::string a = "hello";
std::string b = a;
```

copy 之後，你期待：

```cpp
a += " A";
b += " B";
```

這兩個 string 應該都能正常使用。

scope 結束時：

```text
a destructor runs
b destructor runs
```

它們也都應該能正常銷毀。

你不需要知道 `std::string` 內部是不是有 small string optimization、heap allocation、reference counting 歷史，或其他實作細節。

你只需要知道：

```text
std::string` 的 copy operation 必須讓 copy 後的兩個 string 都仍然是正常 string。
```

這就是 type operation 的重點。

呼叫端不應該直接關心：

```text
std::string 裡面有哪些 bytes？
那些 bytes 要怎麼複製？
內部 pointer 要不要指向同一塊 memory？
```

呼叫端真正依賴的是：

```text
copy 後的 a 和 b 都能像 string 一樣被使用和銷毀。
```

## Why This Matters

回到上一章：

```cpp
T make() {
    T x;
    return x;
}

T y = make();
```

如果這段 code 真的需要 copy，那問題不是只有：

```text
copy 快不快？
```

更根本的問題是：

```text
T 的 copy operation 是否能產生一個正常的 T？
```

這就是為什麼 C++ 不能只把 object 當成一包 bytes。

對某些 type：

```text
copy 很自然。
copy 便宜。
copy 就是複製幾個 fields。
```

對某些 type：

```text
copy 正確，但昂貴。
```

對某些 type：

```text
copy 看起來便宜，但其實語意錯誤。
```

對某些 type：

```text
copy 根本不應該存在。
```

這些差異不是 compiler 猜出來的。

它們應該由 type 的 operations 表達出來。

## The Pressure Point

現在可以問下一個問題：

```text
什麼 type 會讓 byte-copy mental model 直接壞掉？
```

最小例子通常是 resource-owning type：

```c
typedef struct {
    char* ptr;
    size_t size;
} Buffer;
```

如果你寫：

```c
Buffer a = buffer_create(1024);
Buffer b = a;
```

這行看起來只是 copy。

但它會立刻逼出一串問題：

```text
a 和 b 是否指向同一塊 heap memory？
copy 後誰 owns 那塊 memory？
destroy a 時會不會影響 b？
destroy b 時會不會 double free？
```

這就是下一章要處理的地方。

## What This Chapter Does Not Explain Yet

這章還沒有正式講：

```text
move
std::move
RVO
NRVO
copy elision
Rule of 3 / 5
RAII
```

因為在理解 move 之前，必須先知道：

```text
copy 不是只問「貴不貴」。
copy 先問「這個操作到底有沒有正確意義」。
```

## Three Things To Take Away

1. 對簡單 type，copy 看起來可以只是複製 fields，但這不是一般 C++ object 的完整模型。
2. copy 後的 object 必須能正常使用、正常銷毀，而不是只是 bytes 看起來相同。
3. 一旦 type 擁有 resource，copy 就必須回答 ownership、lifetime、destroy 這些問題。

## Next Question

```text
如果一個 type 裡面有 raw pointer 指向 heap memory，
直接 copy 會發生什麼事？
```

Next:

- [[20-languages/cpp/teaching/Chapter 3 - C Buffer Representation Copy Is Not Semantic Copy]]
