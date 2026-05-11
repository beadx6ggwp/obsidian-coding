# Chapter 5 - Cpp Buffer Copy Destroy Become Type Semantics

Related:

- [[20-languages/cpp/Learning Path - C++ Object and Resource Semantics]]
- [[20-languages/cpp/teaching/Chapter 4 - Deep Copy Deleted Copy And Type Design Pressure]]
- [[20-languages/cpp/object-resource-semantics/Concept - RAII and Rule of 0 3 5]]
- [[20-languages/cpp/deep-dives/Deep Dive - Buffer Bug RAII Rules and noexcept Move]]

## Goal

前幾章都在逼同一個問題：

```text
Buffer 擁有 heap memory。

那它：
    怎麼建立 resource？
    怎麼釋放 resource？
    能不能 copy？
    如果能 copy，要怎麼 copy？
    如果不能 copy，要怎麼禁止？
```

這章開始把這些問題放進 C++ type 裡。

重點不是說：

```text
C++ class 比 C struct 好看。
```

而是：

```text
constructor / destructor / copy operation
開始成為 type 表達 ownership 規則的地方。
```

## From C Convention To C++ Type Operations

在 C-style API 裡，我們可能寫：

```c
Buffer a = buffer_create(1024);
buffer_destroy(&a);
```

這裡的規則散在 function name 和 convention 裡：

```text
buffer_create:
    建立一個可用的 Buffer

buffer_destroy:
    釋放 Buffer 擁有的 resource

Buffer b = a:
    這件事到底能不能做，要看文件或 convention
```

C++ 會把一部分規則拉進 type operation：

```cpp
class Buffer {
public:
    explicit Buffer(size_t n);
    ~Buffer();

private:
    char* ptr;
    size_t size;
};
```

現在至少可以看出：

```text
Buffer(size_t n):
    建立 object 時取得 resource

~Buffer():
    object 結束 lifetime 時釋放 resource
```

這就是 RAII 的核心直覺：

```text
resource lifetime follows object lifetime.
```

先不要把 RAII 當成口號。

就把它看成：

```text
我不想讓 create / destroy 規則散在外面。
我希望 object 自己知道它什麼時候取得 resource、什麼時候釋放 resource。
```

## A First C++ Buffer

先寫一個最直接的版本：

```cpp
class Buffer {
public:
    explicit Buffer(size_t n)
        : ptr(new char[n]), size(n) {}

    ~Buffer() {
        delete[] ptr;
    }

private:
    char* ptr;
    size_t size;
};
```

看起來這已經比 C-style API 好很多。

使用時：

```cpp
void work() {
    Buffer a(1024);

    // scope 結束時，~Buffer() 自動執行
}
```

你不需要手動寫：

```cpp
buffer_destroy(&a);
```

這解決了一部分 lifetime 問題。

但它還沒有解決 copy 問題。

## The Hidden Bug Is Still There

如果你寫：

```cpp
Buffer a(1024);
Buffer b = a;
```

會發生什麼？

很多人會以為：

```text
class 有 destructor 了。
C++ 應該會自動把 copy 處理好。
```

這是錯的。

如果你沒有自己定義 copy constructor，compiler 仍然可能產生一個 memberwise copy：

```text
b.ptr  = a.ptr
b.size = a.size
```

也就是：

```text
a.ptr ----+
          +----> same heap block
b.ptr ----+
```

這和前面的 C `Buffer b = a` 本質上是同一個問題。

scope 結束時：

```text
~Buffer() for b -> delete[] b.ptr
~Buffer() for a -> delete[] a.ptr
```

如果 `a.ptr` 和 `b.ptr` 指向同一塊 memory，就可能 double delete。

所以這個版本只說清楚了：

```text
destructor releases resource.
```

但還沒說清楚：

```text
copy creates what relationship between a and b?
```

## Destructor Creates Copy Pressure

這是一個很重要的轉折。

當你寫：

```cpp
~Buffer() {
    delete[] ptr;
}
```

你其實在告訴讀者：

```text
這個 object owns ptr。
object 死掉時會釋放 ptr。
```

那下一個問題一定會跟著出現：

```text
如果這個 object 被 copy，
兩個 object 誰 owns ptr？
```

所以 destructor 不是孤立的東西。

它會對 copy operation 產生壓力。

這就是為什麼後面會有 Rule of 3 / 5。

但現在先不要背規則。

先抓住更基本的因果：

```text
如果 destructor 代表 ownership，
copy 就必須一起定義 ownership 如何處理。
```

## Option A - Define Deep Copy

如果我們希望 `Buffer` 可以 copy，而且 copy 後兩邊各自擁有一份獨立 memory，那就要定義 copy constructor。

概念上：

```cpp
class Buffer {
public:
    explicit Buffer(size_t n)
        : ptr(new char[n]), size(n) {}

    ~Buffer() {
        delete[] ptr;
    }

    Buffer(const Buffer& other)
        : ptr(new char[other.size]), size(other.size) {
        std::copy(other.ptr, other.ptr + other.size, ptr);
    }

private:
    char* ptr;
    size_t size;
};
```

現在：

```cpp
Buffer a(1024);
Buffer b = a;
```

代表：

```text
a.ptr ----> heap block A
b.ptr ----> heap block B
```

`a` 和 `b` 可以各自使用、各自銷毀。

這時 copy constructor 不是效能優化。

它是在定義：

```text
copy Buffer means allocate another buffer and copy the content.
```

也就是：

```text
copy 的意義由 Buffer 這個 type 定義。
```

## Copy Assignment Is Another Operation

還有另一種 copy：

```cpp
Buffer a(1024);
Buffer b(2048);

b = a;
```

這不是 copy construction。

這是 copy assignment。

它更麻煩，因為 `b` 原本已經擁有一塊 memory。

如果要讓它支援 deep copy，大概需要處理：

```text
1. 配置新的 memory
2. 複製 a 的內容
3. 釋放 b 原本的 memory
4. 讓 b 指向新 memory
5. 處理 self-assignment
6. 處理 allocation failure
```

一個簡化版本可能長這樣：

```cpp
Buffer& operator=(const Buffer& other) {
    if (this == &other) {
        return *this;
    }

    char* new_ptr = new char[other.size];
    std::copy(other.ptr, other.ptr + other.size, new_ptr);

    delete[] ptr;

    ptr = new_ptr;
    size = other.size;

    return *this;
}
```

這段不是要你現在背起來。

它只是要讓你看到：

```text
copy assignment 也不是 compiler 小事。
```

只要 type owns resource，assignment 就必須維持同一個 ownership rule。

## Option B - Delete Copy

如果這個 `Buffer` 的設計不是「copy 就複製一整塊 memory」，而是「唯一擁有一塊 buffer」，那更誠實的做法可能是禁止 copy：

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

private:
    char* ptr;
    size_t size;
};
```

現在這段 code 會被拒絕：

```cpp
Buffer a(1024);
Buffer b = a; // error: copy constructor is deleted
```

這很重要。

因為 type 現在明確說：

```text
copy is not a valid operation for this Buffer.
```

這比讓 shallow copy 偷偷發生，然後 runtime double delete，好太多。

這裡的 `= delete` 不是語法裝飾。

它是在把原本靠 convention 的規則放進 type interface：

```text
這個 object 可以被建立。
這個 object 會釋放 resource。
這個 object 不可以被 copy。
```

## What Changed Compared With C?

在 C 裡，規則可能是：

```text
請不要直接寫 Buffer b = a。
要 copy 請用 buffer_clone。
用完請記得 buffer_destroy。
```

這些都可以寫在文件裡。

也可以靠團隊紀律做到。

但 C++ type operation 可以讓一部分規則變成語言直接檢查的事情：

```text
constructor:
    建立 usable object

destructor:
    釋放 owned resource

copy constructor:
    定義 copy construction 的意義，或禁止它

copy assignment:
    定義 assignment 的意義，或禁止它
```

這就是這章標題的意思：

```text
copy / destroy become type semantics.
```

它們不只是 function。

它們是在回答：

```text
這個 type 的 object 可以怎麼被建立、複製、指派、銷毀？
```

## Why This Still Is Not The End

如果我們選 deep copy：

```text
copy 正確，但可能昂貴。
```

如果我們選 delete copy：

```text
copy 被禁止，但 object 變得不好交付。
```

例如：

```cpp
Buffer make_buffer();
std::vector<Buffer> buffers;
```

如果 `Buffer` 不能 copy，這些 value-style API 還能不能好用？

這就逼出下一個問題：

```text
如果 source object 可以被放棄，
能不能把 resource ownership 轉移出去？
```

這才是 move 的出場點。

## What This Chapter Does Not Explain Yet

這章還不正式講：

```text
move constructor
move assignment
std::move
xvalue
moved-from state
RVO
```

這些下一段才會開始出現。

這章只要記住：

```text
一旦 type owns resource，
constructor / destructor / copy constructor / copy assignment
就不再只是語法細節。

它們是 type 用來維護 ownership rule 的 operations。
```

## Three Things To Take Away

1. 只有 destructor 不夠；如果 type owns resource，compiler-generated copy 仍然可能造成 shallow copy 和 double delete。
2. 如果 `Buffer` 支援 copy，copy constructor / copy assignment 必須定義 deep copy、shared ownership，或其他明確規則。
3. 如果 copy 不符合 type 的意思，`= delete` 可以把「不能 copy」變成 type interface，而不是只靠 convention。

## Next Question

```text
如果 copy 被禁止，或 deep copy 太貴，
那 C++ 還能不能保留好用的 value-style API？
```

Next:

- [[20-languages/cpp/teaching/Chapter 6 - Why Move Exists]]
