# Chapter 6 - Why Move Exists

Related:

- [[20-languages/cpp/Learning Path - C++ Object and Resource Semantics]]
- [[20-languages/cpp/teaching/Chapter 5 - Cpp Buffer Copy Destroy Become Type Semantics]]
- [[20-languages/cpp/object-resource-semantics/Concept - Move Semantics and Ownership]]
- [[20-languages/cpp/deep-dives/Deep Dive - Move Ownership Matrix4x4 and Resource Transfer]]

## Goal

前幾章一路把問題推到這裡：

```text
shallow copy 可能錯。
deep copy 可能貴。
deleted copy 可能是正確設計。
```

但如果一個 type 不能 copy，C++ 還能不能保留好用的 value-style API？

例如：

```cpp
Buffer make_buffer();
std::vector<Buffer> buffers;
```

這章才正式讓 move 出場。

重點不是：

```text
move 是比較快的 copy。
```

而是：

```text
copy means duplication.
move means transfer.
```

## The Problem With Only Copy Or No Copy

假設我們把 `Buffer` 設計成 unique owner：

```cpp
class Buffer {
public:
    explicit Buffer(size_t n);
    ~Buffer();

    Buffer(const Buffer&) = delete;
    Buffer& operator=(const Buffer&) = delete;

private:
    char* ptr;
    size_t size;
};
```

這很合理。

因為：

```text
shallow copy 會 double delete。
deep copy 可能不是這個 type 想表達的操作。
```

所以 type 說：

```text
I cannot be copied.
```

但現在你會遇到另一種壓力。

如果有一個 function 建立並回傳 `Buffer`：

```cpp
Buffer make_buffer() {
    Buffer b(1024);
    return b;
}
```

或你想把一個 `Buffer` 放進某個 owner：

```cpp
std::vector<Buffer> buffers;
```

如果 `Buffer` 完全不能 copy，也不能用其他方式轉移，那很多 value-style 寫法都會變得很難用。

這就是 move 要解的問題。

## The Missing Operation

現在假設有一個 object `a`：

```cpp
Buffer a(1024);
```

它 owns 一塊 heap memory：

```text
a.ptr ----> heap block
```

如果我們要讓 `b` 得到一個 `Buffer`，有幾種可能？

第一種是 deep copy：

```text
a.ptr ----> heap block A
b.ptr ----> heap block B
```

這表示：

```text
a 和 b 都保有完整資料。
```

但如果 `a` 之後不再需要原本那塊 memory 呢？

例如：

```text
a 是一個暫時結果。
a 是 function 裡快要離開 scope 的 local object。
a 是某個 container 內要被重新安置的 element。
```

這時候我們真正想做的可能不是 duplicate。

而是 transfer：

```text
before:
    a.ptr ----> heap block
    b.ptr ----> nothing

after:
    b.ptr ----> heap block
    a.ptr ----> empty / null / safe state
```

這就是 move 的核心。

## Copy vs Move

可以先用一句話分開：

```text
copy:
    A 和 B 都要一份。

move:
    A 不再需要原本 resource。
    B 接手 A 的 resource。
```

更精準地說：

```text
copy means duplication.
move means transfer.
```

所以 move 不是 deep copy 的魔法加速版。

它回答的是不同問題：

```text
copy:
    如何讓 B 得到一份自己的內容？

move:
    如果 A 可以放棄原本內容，
    B 能不能接手 A 的內容？
```

如果 A 和 B 都需要完整資料：

```text
你需要 copy。
```

如果 A 之後還要繼續保有原本內容：

```text
你不能 move from A。
```

如果 A 可以被放棄：

```text
move 才是合理語意。
```

## A Move Constructor Sketch

概念上，`Buffer` 的 move constructor 可能像這樣：

```cpp
class Buffer {
public:
    explicit Buffer(size_t n)
        : ptr(new char[n]), size(n) {}

    ~Buffer() {
        delete[] ptr;
    }

    Buffer(const Buffer&) = delete;
    Buffer& operator=(const Buffer&) = delete;

    Buffer(Buffer&& other)
        : ptr(other.ptr), size(other.size) {
        other.ptr = nullptr;
        other.size = 0;
    }

private:
    char* ptr;
    size_t size;
};
```

這段最重要的是：

```cpp
ptr = other.ptr;
size = other.size;
```

destination 接手 source 的 resource。

然後：

```cpp
other.ptr = nullptr;
other.size = 0;
```

source 被放到一個可以安全 destroy 的狀態。

現在 scope 結束時：

```text
destination destructor:
    delete[] heap block

source destructor:
    delete[] nullptr
```

不會 double delete。

## Moved-From Is Not Destroyed

這裡有一個很重要的點：

```text
被 move 的 object 還活著。
```

Move 不是 destructor。

Move 也不是把 object 從世界上移除。

如果我們有：

```cpp
Buffer b = /* move from a */;
```

move 之後：

```text
b:
    owns the original heap block

a:
    still exists
    no longer owns that heap block
    must remain safe to destroy
```

這就是為什麼 move constructor 必須把 source 放到一個安全狀態。

現在先不要急著把這叫做完整的 `moved-from state` 規則。

先抓住：

```text
source object 沒有死。
source object 只是放棄了某些 resource。
source object 之後仍然會被 destructor 處理。
```

## Why C++11 Needed This As A Language Feature

在 C++11 以前，大家不是完全不能做 transfer。

你可以用：

```text
raw pointer handoff
swap trick
out parameter
auto_ptr-style destructive copy
manual ownership convention
```

但這些做法有一個共同問題：

```text
transfer 不是乾淨的一等 type operation。
```

C++11 move semantics 的重點是：

```text
讓 type 可以正式定義：
    copy 是什麼；
    move 是什麼；
    什麼時候可以 transfer resource。
```

這樣 library 和 compiler 才能在適當情況下選用 move operation。

所以更精準的說法不是：

```text
C++11 發明 move 是因為 deep copy 很慢。
```

而是：

```text
C++11 讓 resource transfer 成為 type system 和 overload resolution 可以表達的操作。
```

Deep copy expensive 是重要動機之一。

但更大的問題是：

```text
有些 object 不該 copy；
但仍然需要被交付。
```

## Why This Preserves Value-Style APIs

C++ 很重視 value-style API。

你希望可以寫：

```cpp
Buffer make_buffer();
Buffer b = make_buffer();
```

你也希望 resource-owning object 能放進 container、從 function return、被重新配置。

如果只能 copy：

```text
resource owner 不是昂貴 deep copy，
就是根本不能走很多 value-style 路徑。
```

Move 讓 C++ 有一條中間路：

```text
不 duplicate resource。
而是在 source 可被放棄時 transfer resource。
```

所以 move 的價值是：

```text
保留 value-style API，
同時保留 resource ownership 的正確性。
```

## What This Chapter Does Not Explain Yet

這章還沒有解釋：

```text
std::move
xvalue
rvalue reference
overload resolution
return x vs return std::move(x)
RVO / NRVO
```

你現在只需要先知道：

```text
move constructor 是真正做 transfer 的 operation。
std::move 不是。
```

下一章才會問：

```text
那 `std::move(x)` 到底做了什麼？
```

## Three Things To Take Away

1. Move 不是 faster copy；copy 是 duplication，move 是 transfer。
2. Move 只有在 source object 可以被放棄時才合理；如果 A 和 B 都需要完整資料，deep copy 才是正確語意。
3. Move constructor 接手 resource 後，source object 仍然活著，所以必須被放在可以安全 destroy 的狀態。

## Next Question

```text
如果 move constructor 才真正 transfer resource，
那 `std::move(x)` 到底做了什麼？
```

Next:

- [[20-languages/cpp/teaching/Chapter 7 - Move Is Ownership Transfer Not Faster Copy]]
