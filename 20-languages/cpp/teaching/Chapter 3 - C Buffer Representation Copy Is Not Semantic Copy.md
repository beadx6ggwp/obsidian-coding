# Chapter 3 - C Buffer Representation Copy Is Not Semantic Copy

Related:

- [[20-languages/cpp/Learning Path - C++ Object and Resource Semantics]]
- [[20-languages/cpp/teaching/Chapter 2 - Copy Is Not Just Moving Bytes]]
- [[20-languages/cpp/conversation-notes/Conversation Note - C vs C++ Semantic Lifting and Hidden Semantics]]
- [[20-languages/cpp/deep-dives/Deep Dive - C Convention to Cpp Semantic Lifting]]

## Goal

上一章先建立了一個壓力：

```text
copy 不是只問 bytes 像不像。
copy 後的 object 要能正常使用、正常銷毀。
```

這章用一個 C-style `Buffer` 讓這件事變得具體。

重點不是說 C 做不到。

重點是：

```text
C 可以靠 convention 做對。
但只看 struct layout 和 assignment，
你看不出 copy 的真正意義。
```

## Cold Open

先看一個很常見的 C-style resource wrapper：

```c
typedef struct {
    char* ptr;
    size_t size;
} Buffer;
```

假設我們有兩個 helper functions：

```c
Buffer buffer_create(size_t size);
void buffer_destroy(Buffer* buffer);
```

使用時可能長這樣：

```c
Buffer a = buffer_create(1024);
Buffer b = a;

buffer_destroy(&a);
buffer_destroy(&b);
```

這段看起來很短。

但它其實藏了很多問題。

## The Naive Picture

如果用上一章的 naive model，你可能會說：

```text
Buffer b = a;
```

就是 copy 一份 `Buffer`。

也就是：

```text
b.ptr  = a.ptr
b.size = a.size
```

從 representation 的角度，這句話沒錯。

`Buffer` 裡面有兩個 fields：

```text
char* ptr
size_t size
```

所以 assignment 會把這兩個 fields 複製過去。

問題是：

```text
複製 representation，不代表複製了正確的 ownership meaning。
```

## What Actually Gets Copied

假設 `buffer_create(1024)` 在 heap 上配置一塊 memory：

```text
a.ptr ----> heap block
a.size = 1024
```

執行：

```c
Buffer b = a;
```

之後，通常會變成：

```text
a.ptr ----+
          +----> same heap block
b.ptr ----+

a.size = 1024
b.size = 1024
```

這叫 shallow copy。

不是因為 compiler 做錯。

而是因為 C struct assignment 的行為就是複製 fields。

它不知道：

```text
ptr 是 owner 嗎？
ptr 只是 view 嗎？
ptr 指向 single char 還是一段 buffer？
copy 後要不要配置新的 heap block？
copy 後 a 和 b 能不能各自 destroy？
```

這些問題不在 `typedef struct` 裡。

它們在 programmer convention 裡。

## The Double Free Problem

現在看最後兩行：

```c
buffer_destroy(&a);
buffer_destroy(&b);
```

如果 `buffer_destroy` 大概做的是：

```c
void buffer_destroy(Buffer* buffer) {
    free(buffer->ptr);
    buffer->ptr = NULL;
    buffer->size = 0;
}
```

那第一個 destroy：

```c
buffer_destroy(&a);
```

會釋放 heap block。

但 `b.ptr` 還指向同一塊 memory：

```text
a.ptr ----> NULL

b.ptr ----> freed heap block
```

接著：

```c
buffer_destroy(&b);
```

就可能再次 `free` 同一塊 memory。

這就是 double free。

問題不是：

```text
C 完全不能管理 resource。
```

問題是：

```text
`Buffer b = a` 這個語法沒有告訴你：
copy 後 ownership 應該怎麼處理。
```

## The Hidden Questions

只看這個 type：

```c
typedef struct {
    char* ptr;
    size_t size;
} Buffer;
```

你其實不知道：

```text
Buffer 能不能 copy？
如果能 copy，是 shallow copy 還是 deep copy？
如果 shallow copy，兩個 Buffer 是共享同一塊 memory 嗎？
如果 shared，誰負責最後 free？
如果 deep copy，copy 時誰負責配置新的 memory？
如果不能 copy，要怎麼防止別人寫 Buffer b = a？
Buffer 的 lifetime 從哪裡開始？
Buffer 的 lifetime 到哪裡結束？
ptr 什麼時候是 valid pointer？
ptr 什麼時候可以是 NULL？
```

這些才是真正重要的東西。

但 C 的 struct layout 只告訴你：

```text
Buffer 裡面放了哪些資料。
```

它沒有直接告訴你：

```text
這些資料代表什麼 ownership rule。
```

## C Can Still Do It

這裡要小心，不要把問題講成：

```text
C 爛，所以 C++ 好。
```

C 當然可以把這件事做好。

例如你可以規定：

```text
Buffer 不可以直接 assignment。
只能透過 buffer_clone 做 deep copy。
只能透過 buffer_destroy 釋放。
```

你也可以寫：

```c
Buffer buffer_clone(const Buffer* src);
```

讓 copy 明確變成：

```c
Buffer b = buffer_clone(&a);
```

這樣比直接：

```c
Buffer b = a;
```

清楚得多。

但注意差異：

```text
正確性依賴命名、文件、團隊規範、code review、使用者記得不要做錯。
```

也就是：

```text
meaning lives in convention.
```

這就是這章真正要抓的點。

## Representation Copy vs Meaningful Copy

現在回到上一章的問題：

```text
copy 之後，
B 應該是一個可以正常使用、正常銷毀的 T。
```

對 `Point`：

```cpp
Point b = a;
```

通常沒有問題。

對 C-style `Buffer`：

```c
Buffer b = a;
```

representation copy 也確實發生了。

但這個 copy 沒有回答：

```text
B 是否真的可以正常使用？
B 是否真的可以正常銷毀？
A 和 B 是否都認為自己 owns 同一塊 memory？
```

所以這裡可以先得到一句很重要的話：

```text
Representation copy is not semantic copy.
```

更口語地說：

```text
fields 複製過去了，
但「誰負責這塊 resource」沒有被說清楚。
```

## Why This Matters For C++

C++ 的重要方向不是把這段 C code 寫得比較漂亮。

也不是單純把：

```c
buffer_create(...)
buffer_destroy(...)
```

換成：

```cpp
Buffer(...)
~Buffer()
```

真正重要的是：

```text
copy / destroy / later move
都會變成這個 type 自己要表達的 operations。
```

也就是說，C++ 會開始問：

```text
Buffer 能不能 copy？
如果能 copy，要怎麼 copy？
如果不能 copy，要怎麼禁止？
如果 object 死掉，要怎麼釋放 resource？
```

但在正式進 C++ class 之前，還有一個中間問題值得先看：

```text
如果 shallow copy 會錯，
那正確的 copy 可能有哪些選擇？
```

## What This Chapter Does Not Explain Yet

這章還不講：

```text
constructor
destructor
RAII
Rule of 3 / 5
move
std::move
RVO
```

因為現在重點還不是 C++ 的完整解法。

現在重點只是看清楚問題：

```text
copy 不只是複製 fields。
copy 必須處理 resource ownership。
```

## Three Things To Take Away

1. C struct assignment 會複製 representation，但不會替你定義 ownership rule。
2. `Buffer b = a` 可能讓 `a.ptr` 和 `b.ptr` 指向同一塊 heap memory，導致 double free 風險。
3. C 可以靠 convention 解決這件事，但只看 struct layout 和 assignment，看不出 copy 的真正意義。

## Next Question

```text
如果 shallow copy 會錯，
那正確 copy 是 deep copy 嗎？
還是這個 type 根本不應該允許 copy？
```

Next:

- [[20-languages/cpp/teaching/Chapter 4 - Deep Copy Deleted Copy And Type Design Pressure]]
