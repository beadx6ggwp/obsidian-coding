# Chapter 4 - Deep Copy Deleted Copy And Type Design Pressure

Related:

- [[20-languages/cpp/Learning Path - C++ Object and Resource Semantics]]
- [[20-languages/cpp/teaching/Chapter 3 - C Buffer Representation Copy Is Not Semantic Copy]]
- [[20-languages/cpp/object-resource-semantics/Concept - RAII and Rule of 0 3 5]]
- [[20-languages/cpp/deep-dives/Deep Dive - Buffer Bug RAII Rules and noexcept Move]]

## Goal

上一章看到：

```c
Buffer a = buffer_create(1024);
Buffer b = a;
```

這行看起來只是 copy。

但如果 `Buffer` owns heap memory，這個 copy 可能只是把 pointer value 複製過去，最後造成 double free。

這章要問：

```text
如果 shallow copy 會錯，
那正確的 copy 是什麼？
```

這章仍然不正式講 move。

因為 move 要出現以前，必須先看清楚：

```text
copy 有幾種可能設計；
有些正確但昂貴；
有些便宜但錯；
有些 type 根本不應該允許 copy。
```

## The Design Question

回到這個 type：

```c
typedef struct {
    char* ptr;
    size_t size;
} Buffer;
```

如果我們寫：

```c
Buffer b = a;
```

我們不能只問：

```text
這行會不會複製 bytes？
```

我們要問：

```text
copy 後，a 和 b 應該是什麼關係？
```

這個問題至少有幾種可能答案：

```text
1. a 和 b 指向同一塊 memory
2. a 和 b 各自擁有一份獨立 memory
3. 這個 type 根本不允許 copy
```

這三個答案都不是單純的 syntax choice。

它們是 type design choice。

## Option 1 - Shallow Copy

Shallow copy 的意思是：

```text
a.ptr ----+
          +----> same heap block
b.ptr ----+
```

它通常很便宜，因為只是複製 pointer 和 size：

```text
b.ptr  = a.ptr
b.size = a.size
```

但便宜不代表正確。

如果 `Buffer` 的意思是：

```text
這個 object 唯一擁有這塊 heap memory，
object 死掉時會 free 它。
```

那 shallow copy 會讓兩個 object 都像 owner：

```text
a thinks it owns the heap block
b thinks it owns the heap block
```

最後就可能：

```text
destroy a -> free heap block
destroy b -> free same heap block again
```

所以對 unique ownership 的 `Buffer` 來說：

```text
shallow copy is cheap,
but it probably gives the wrong meaning.
```

## Caveat: Shallow Copy Is Not Always Wrong

這裡要小心。

Shallow copy 不是在所有情境都錯。

如果 type 的意思本來就是：

```text
view
borrowed pointer
shared handle
reference-counted object
```

那多個 object 指向同一個 resource 可能是合理的。

例如：

```text
string_view:
    copy view, not the string data

shared_ptr:
    copy shared ownership, not the pointed object

span:
    copy pointer + length view, not the array
```

重點不是 shallow copy 本身邪惡。

重點是：

```text
copy behavior must match the type's meaning.
```

如果 `Buffer` 是 owner，shallow copy 很可能錯。

如果 `BufferView` 是 view，shallow copy 可能完全合理。

名字、API、operation 都要一起說清楚。

## Option 2 - Deep Copy

Deep copy 的意思是：

```text
a.ptr ----> heap block A

b.ptr ----> heap block B
```

也就是 copy 時配置新的 memory，並把內容複製過去。

概念上可能像這樣：

```c
Buffer buffer_clone(const Buffer* src) {
    Buffer dst = buffer_create(src->size);
    memcpy(dst.ptr, src->ptr, src->size);
    return dst;
}
```

使用時：

```c
Buffer a = buffer_create(1024);
Buffer b = buffer_clone(&a);

buffer_destroy(&a);
buffer_destroy(&b);
```

這樣 `a` 和 `b` 各自擁有自己的 heap block。

所以：

```text
destroy a 不會破壞 b。
destroy b 不會 double free a 的 memory。
```

這通常是「copy 後兩邊都要各自保有完整資料」時的正確語意。

但它有成本：

```text
allocate new memory
copy all bytes
可能失敗
可能很慢
```

所以 deep copy 的狀態是：

```text
semantically correct,
but possibly expensive.
```

這也更精準地修正了「copy 很貴」這個直覺。

不是所有 copy 都貴。

而是：

```text
如果正確 copy 必須 duplicate resource，
那它可能很貴。
```

## Option 3 - Deleted Copy

有些 type 的正確設計不是 deep copy。

而是：

```text
不要允許 copy。
```

例如一個 type 表示：

```text
唯一擁有一個 resource。
```

那 copy 可能不符合它的概念。

如果兩個 object 都擁有同一個 resource，會錯。

如果 deep copy 一份 resource，又不符合設計意圖，或成本不可接受。

那最誠實的設計可能是：

```text
this type is not copyable.
```

在 C 裡，你很難直接禁止：

```c
Buffer b = a;
```

所以通常只能靠 convention、opaque type、API design 或 code review。

在 C++ 裡，後面會看到你可以把這件事寫進 type：

```cpp
Buffer(const Buffer&) = delete;
Buffer& operator=(const Buffer&) = delete;
```

這不是語法炫技。

它是在說：

```text
這個 type 沒有 copy 這個操作。
```

也就是：

```text
deleted copy can be the honest type operation.
```

## Why This Creates Pressure

現在我們有三種選擇：

```text
shallow copy:
    cheap, but wrong for unique ownership.

deep copy:
    correct if independent duplication is desired,
    but may be expensive.

deleted copy:
    honest if duplication is not a valid operation,
    but now the object cannot be copied.
```

這裡開始出現真正的設計壓力。

如果 copy 被 delete：

```text
那我要怎麼把 Buffer 放進 container？
我要怎麼從 function return Buffer？
我要怎麼把 Buffer 從一個 object 交給另一個 object？
```

如果 deep copy 很貴：

```text
那每次 function return / container reallocation / assignment
都要複製整塊 heap memory 嗎？
```

這些問題不是小優化。

它們會影響 C++ 能不能維持一種好用的 value-style API。

## The Key Distinction

到這裡，可以把前幾章收束成一句話：

```text
copy means duplication.
```

但 duplication 有不同情況：

```text
Point:
    duplicate two ints.

string:
    duplicate string content enough that both strings are usable.

unique Buffer:
    duplication might require deep copy,
    or might not be allowed at all.
```

所以你的直覺：

```text
deep copy 很吃效能，所以如果能轉移會更好。
```

方向是對的，但還要補一個條件：

```text
只有當 source object 可以被放棄時，
transfer 才合理。
```

如果 `a` 和 `b` 都需要完整資料：

```text
deep copy 才是正確語意。
```

如果 `a` 不再需要原本 resource：

```text
那才開始有「能不能轉移 ownership」這個問題。
```

這就是 move 的前置動機。

## What This Chapter Does Not Explain Yet

這章還不正式講：

```text
move constructor
std::move
xvalue
moved-from state
RVO
NRVO
```

因為 move 不是一開始就該出現的答案。

move 是在這些壓力之後才自然出現：

```text
copy 可以很貴。
copy 可以語意錯誤。
copy 可以根本不應該存在。
但我們仍然想把 object 從一個地方交到另一個地方。
```

## Three Things To Take Away

1. `shallow copy` 不是永遠錯，但對 unique ownership 的 resource owner 通常是錯的。
2. `deep copy` 可以是正確 copy，但如果 resource 很大或配置昂貴，就可能很貴。
3. 有些 type 最誠實的設計是禁止 copy，這會自然逼出下一個問題：不能 copy 的 object 要怎麼被交付？

## Next Question

```text
C++ 能不能讓 Buffer 這個 type 自己說清楚：
它怎麼建立 resource？
它怎麼釋放 resource？
它能不能 copy？
如果不能 copy，要怎麼禁止？
```

Next:

- [[20-languages/cpp/teaching/Chapter 5 - Cpp Buffer Copy Destroy Become Type Semantics]]
