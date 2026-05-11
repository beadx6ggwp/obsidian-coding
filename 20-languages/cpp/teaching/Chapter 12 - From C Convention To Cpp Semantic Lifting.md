# Chapter 12 - From C Convention To Cpp Semantic Lifting

Related:

- [[20-languages/cpp/Learning Path - C++ Object and Resource Semantics]]
- [[20-languages/cpp/teaching/Chapter 3 - C Buffer Representation Copy Is Not Semantic Copy]]
- [[20-languages/cpp/teaching/Chapter 5 - Cpp Buffer Copy Destroy Become Type Semantics]]
- [[20-languages/cpp/teaching/Chapter 11 - return x vs return std move x]]
- [[20-languages/cpp/conversation-notes/Conversation Note - C vs C++ Semantic Lifting and Hidden Semantics]]
- [[20-languages/cpp/deep-dives/Deep Dive - C Convention to Cpp Semantic Lifting]]
- [[20-languages/cpp/object-resource-semantics/Concept - C vs C++ Semantic Lifting]]

## Goal

先不要急著說：

```text
C vs C++
```

先問一個更直接的問題：

```text
為什麼 C++ 需要 copy constructor、move constructor、destructor、
deleted operations、value categories、RVO / copy elision？
```

如果只是把它們當成語法規則背起來，它們會看起來很零碎：

```text
return by value
copy
deep copy / deleted copy
move
std::move
value category
RVO / NRVO
```

但前面每一章其實都在逼近同一件事：

```text
object operation 不是純機械動作。
object operation 會承載語意。
```

例如：

```text
copy:
    不是單純 bytes copy，
    而是產生另一個可正常使用、可正常銷毀的 T。

move:
    不是 faster copy，
    而是 source 可以被放棄時的 ownership transfer。

destructor:
    不是普通 function call，
    而是 object lifetime 結束時 resource 如何收尾。

deleted copy:
    不是少一個方便功能，
    而是 type 明確說「這個操作不符合我的語意」。

RVO / copy elision:
    不是單純 compiler trick，
    而是 object 可以直接在結果位置形成時，
    transfer 本身就不需要。
```

所以這章的起點不是：

```text
C++ 比 C 好。
```

而是：

```text
為什麼這些 object operations 需要存在？
它們到底在保存什麼語意？
```

接著才問下一步：

```text
如果 type operation / object lifetime 不承載這些語意，
那語意會去哪裡？
```

答案通常是：

```text
API convention
comments
documentation
naming
programmer discipline
```

這時 C-style code 就是一個很好的對照。

不是因為 C 做不到。

而是因為 C 很清楚地展示：

```text
當語言層的 operation 主要描述 representation manipulation，
ownership / lifetime / copy semantics 就常常要靠 convention 補上。
```

這就是這裡說的：

```text
semantic lifting
```

## Cold Open

現在再看一個 C-style `Buffer`：

```c
typedef struct {
    char* ptr;
    size_t size;
} Buffer;

Buffer buffer_create(size_t n);
void buffer_destroy(Buffer* b);
```

使用端可能這樣寫：

```c
Buffer a = buffer_create(1024);
Buffer b = a;

buffer_destroy(&a);
buffer_destroy(&b);
```

這段 code 的問題不是：

```text
C 做不到 Buffer。
```

C 當然做得到。

問題是，只看這段 code，你不知道：

```text
Buffer b = a;
```

到底是不是合法操作。

## The Hidden Questions

這一行：

```c
Buffer b = a;
```

在 C 的語言層面，很容易被看成：

```text
copy the struct representation
```

也就是：

```text
b.ptr = a.ptr
b.size = a.size
```

但對一個 resource-owning `Buffer` 來說，真正重要的問題不是 bytes 怎麼複製。

真正重要的是：

```text
Buffer owns ptr 嗎？
ptr 指向一個 heap allocation 嗎？
Buffer 能不能 copy？
如果能 copy，是 shallow copy 還是 deep copy？
copy 之後誰負責 free？
a 和 b 的 lifetime 各自怎麼結束？
destroy 之後 Buffer 會變成什麼狀態？
```

這些問題都不是假的。

它們就是寫 C code 時真正要知道的事。

只是它們常常不在 `struct` definition 裡。

它們可能藏在：

```text
function naming
comments
documentation
team convention
caller discipline
reviewer's memory
```

所以這裡的核心句是：

```text
C 的 code 可以完成功能，
但很多正確語意只存在 programmer convention 裡。
```

## This Is Not A C Bad Chapter

這章不是要說：

```text
C 很爛，C++ 比較好。
```

更精準的說法是：

```text
C 把很多 control 交給 programmer。
C++ 試圖讓更多語意能被 type / lifetime / operation 表達出來。
```

C 可以設計更好的 API：

```c
typedef struct Buffer Buffer;

Buffer* buffer_create(size_t n);
void buffer_destroy(Buffer* b);
Buffer* buffer_clone(const Buffer* b);
```

也可以用 opaque type 隱藏 representation，避免 caller 直接做：

```c
Buffer b = a;
```

這些都是 C 裡很合理的工程手段。

但你要注意：

```text
這些仍然主要靠 API discipline 和 convention 維持。
```

C++ 的問題意識不是：

```text
C 完全做不到。
```

而是：

```text
能不能讓 type 自己攜帶更多語意？
```

## What C++ Moves Into The Type

回到 C++ `Buffer`：

```cpp
class Buffer {
public:
    explicit Buffer(std::size_t n);
    ~Buffer();

    Buffer(const Buffer&) = delete;
    Buffer& operator=(const Buffer&) = delete;

    Buffer(Buffer&&) noexcept;
    Buffer& operator=(Buffer&&) noexcept;

private:
    char* ptr;
    std::size_t size;
};
```

這段 code 比 C struct 複雜很多。

但它多出來的東西不是裝飾。

它把原本藏在 convention 裡的問題搬進 type operation：

```text
constructor:
    Buffer 如何取得 resource？

destructor:
    Buffer 死亡時如何釋放 resource？

copy constructor / copy assignment:
    Buffer 能不能 duplicate？
    如果不能，就 delete。

move constructor / move assignment:
    如果 source 可以被放棄，
    ownership 如何 transfer？

private data:
    caller 不能直接改 ptr / size，
    type 可以保護自己的 representation。
```

所以 C++ 不是只把 C code 改寫成 class。

比較深的改變是：

```text
ownership / lifetime / copyability / transfer
被放進 type 的 operation set 裡。
```

## Why Destructor Alone Is Not Enough

前面 Chapter 5 已經看過一個關鍵點。

如果你只寫：

```cpp
class Buffer {
public:
    explicit Buffer(std::size_t n)
        : ptr(new char[n]), size(n) {}

    ~Buffer() {
        delete[] ptr;
    }

private:
    char* ptr;
    std::size_t size;
};
```

這還不夠。

因為 destructor 只回答：

```text
object 死亡時怎麼釋放 resource？
```

但它沒有回答：

```text
object 被 copy 時 resource 怎麼辦？
```

如果 compiler-generated copy 只是 memberwise copy：

```text
a.ptr ----+
          +----> same heap buffer
b.ptr ----+
```

那兩個 destructor 還是可能 double free。

這就是為什麼 RAII 和 copy/move 會連在一起。

不是因為它們是同一個概念。

而是因為它們都在維護同一件事：

```text
resource ownership semantics
```

## How The Earlier Chapters Fit

現在回頭看前面章節。

Chapter 1 從 return by value 開始：

```cpp
T make() {
    T x;
    return x;
}
```

一開始你擔心的是：

```text
會不會 copy？
```

但後來發現真正問題是：

```text
caller 那邊如何得到一個可以正常使用、正常銷毀的 T？
```

Chapter 2 和 3 把 copy 拆開：

```text
copy 不是 bytes movement。
copy 必須符合 type 的意義。
```

`Buffer b = a` 讓你看到：

```text
representation copy 可以發生，
但 ownership semantics 可以壞掉。
```

Chapter 4 和 5 推到 type design：

```text
deep copy:
    如果兩邊都要獨立 resource，這可能是正確語意。

deleted copy:
    如果 duplication 不合理，禁止 copy 才誠實。

type operations:
    copy / destroy / move 必須一起維護 ownership rule。
```

Chapter 6 和 7 引入 move：

```text
copy means duplication.
move means transfer.
```

Move 不是 faster copy。

Move 是 source 可以被放棄時，把 ownership transfer 寫成 type operation。

Chapter 8 和 9 處理 `std::move` / value category：

```text
std::move 不 move。
value category 描述 expression，不是 object lifetime。
```

這些規則看起來很細，但它們的作用是讓 overload resolution 能選到正確 operation：

```text
copy operation
move operation
bind to reference
initialize object
```

Chapter 10 和 11 回到 return by value：

```text
return T{}:
    有時候根本沒有 source object 要 move。

return x:
    保留 NRVO candidate shape。

return std::move(x):
    明確要求從 xvalue transfer，
    但可能放棄 no-transfer path。
```

現在你可以看到，這些不是零散規則。

它們都在回答：

```text
一個 object / resource 的語意如何被建立、保存、轉移、結束？
```

## C Out Parameter vs C++ Return By Value

再看另一個 C 常見寫法：

```c
int buffer_create(Buffer* out, size_t n);
```

這裡有很多 hidden questions：

```text
out 可不可以是 NULL？
成功時 out 是否 initialized？
失敗時 out 是什麼狀態？
caller 什麼時候 destroy？
return int 的 error code 和 out 的 lifetime 怎麼配合？
```

C 可以靠 API contract 說清楚。

但 contract 通常在文件裡。

C++ 會傾向把某些情況寫成：

```cpp
Buffer make_buffer(std::size_t n);
```

或者如果可能失敗：

```cpp
std::optional<Buffer> try_make_buffer(std::size_t n);
```

這不是單純語法變漂亮。

它改變了語意位置：

```text
result 是 return value。
成功時有 Buffer object。
失敗時沒有 Buffer object。
Buffer 的 lifetime 由 object lifetime 管理。
return by value 的成本由 move / copy elision / RVO 模型處理。
```

所以 RVO 也不是孤立最佳化。

它讓 C++ 可以寫出更直接的語意：

```text
this function produces a T
```

而不用因為害怕 copy，退回：

```text
please pass me a pointer to uninitialized output storage
```

## Pointer Semantics Split Apart

C 裡一個 raw pointer 可能同時表示很多意思：

```c
void process(char* p);
```

只看 `char* p`，你不知道：

```text
p 可不可以是 NULL？
p 指向一個 char 還是一段 buffer？
process 會不會修改內容？
process 會不會 free p？
p 的 lifetime 由誰保證？
```

C++ 不是消滅 pointer。

但 C++ library 會提供更多 vocabulary types，把這些語意拆開：

```cpp
void read(std::span<const char> data);
void write(std::span<char> data);
void take(std::unique_ptr<char[]> data);
void view(std::string_view text);
void use(char& c);
```

這些 type 的價值不是「比較潮」。

而是它們讓 function signature 多說了一些原本藏在 convention 裡的話：

```text
span:
    non-owning contiguous view

unique_ptr:
    unique ownership

string_view:
    non-owning string view

reference:
    refers to an existing object, normally not null
```

這就是 semantic lifting 的另一個面向：

```text
用不同 type 表達不同 resource relationship。
```

## What Semantic Lifting Means

現在可以正式定義這個詞。

在這組筆記裡：

```text
semantic lifting =
    把原本只靠 programmer convention 維持的語意，
    提升到 type、object lifetime、language rule、library abstraction 裡。
```

C layer 常常長這樣：

```text
memory
pointer
struct layout
function call
documentation
discipline
```

C++ layer 會試圖加入：

```text
constructor
destructor
copy constructor
move constructor
deleted operation
RAII
value category
return-by-value rules
library vocabulary types
```

不是每一個語意都會被 compiler 完全保證。

C++ 也不是完全 memory-safe language。

但 C++ 給你更多工具，把規則放進程式結構裡，而不是只放在人腦和註解裡。

## Cost Still Matters

這裡還有一個重要限制。

C++ 不只是想把語意表達清楚。

C++ 還想保留接近 C 的成本模型。

所以它才會同時在乎：

```text
copy 是否 deep and expensive
move 是否 cheap transfer
return by value 是否能 elide
object 是否能直接在 final storage constructed
```

這就是為什麼這組筆記會從 RVO 開始。

因為 RVO 正好展示了 C++ 的張力：

```text
我想寫出高階語意：
    function returns T

但我不想付出不必要的 copy / move 成本。
```

C++ 的回答不是單一技巧。

而是一整套 object model：

```text
copy:
    duplicate when duplication is meaningful

move:
    transfer when source can be abandoned

copy elision / RVO:
    construct directly when transfer is unnecessary
```

所以前面那句可以改得更精準：

```text
C++ 不只是在 lifting semantics。
C++ 是在 lifting semantics while preserving a low-level cost model.
```

## The Chapter's Main Claim

現在可以把整條路線收成一句話：

```text
C 可以用 memory、pointer、function、convention 完成很多事。

C++ 的一條核心方向是：
讓 object 的建立、複製、轉移、銷毀，
盡量由 type operations 和 object lifetime 來表達。
```

所以：

```text
constructor / destructor 不是單純語法糖。
copy / move 不是單純效能技巧。
std::move 不是單純把東西搬走。
RVO 不是單純 compiler trick。
```

它們都屬於同一個更大的工程問題：

```text
語意應該存在哪裡？
```

如果語意只存在文件和習慣裡，大型 codebase 很容易漂移。

如果語意能被 type / operation / lifetime 表達，compiler、library、reviewer、使用者都比較容易看見它。

## Bridge To The Next Chapter

到這裡，Chapter 12 其實只回答了一半：

```text
C++ 把很多 hidden convention 提升到哪裡？
```

目前答案是：

```text
提升到 type、object lifetime、type operations、library vocabulary。
```

但這會立刻導出下一個問題：

```text
如果 type 要承載這麼多語意，
那我們還能不能把 type 想成「一段 memory layout + 幾個 function」？
```

以前看 C struct 時，你可能會這樣想：

```text
Buffer =
    char* ptr
  + size_t size
```

也就是把 type 主要理解成 data layout。

但現在 `Buffer` 已經不只是兩個 fields。

它還包含：

```text
constructor:
    什麼狀態算成功建立？

destructor:
    什麼時候釋放 resource？

copy operation:
    能不能複製？如果能，怎麼複製？

move operation:
    ownership 如何轉移？

deleted operation:
    哪些操作根本不應該存在？
```

所以 Chapter 12 的結論不是直接跳到抽象哲學。

它其實逼出下一章的問題：

```text
如果語意被放進 type，
那 type 本身就不可能只是 layout。
```

下一章才正式收束：

```text
Type =
    representation
  + valid states
  + operations
  + invariants
  + lifetime rules
  + cost model
```

## What This Chapter Does Not Explain Yet

這章刻意不展開：

```text
Rust ownership / borrow checker
GC and tracing reachability
concepts / regularity
exception safety
memory model / concurrency
complete type invariant theory
```

這些都跟 semantic lifting 有關，但會把主線帶到其他方向。

下一章只接一個問題：

```text
如果語意被 lifted 到 type，
那 type 到底除了 layout 之外還包含什麼？
```

也就是：

```text
A type is not just a layout.
```

## Three Things To Take Away

1. C 不是做不到 resource management；差別是很多 ownership / lifetime / copy semantics 在 C 裡常靠 convention 維持。
2. C++ 用 constructor、destructor、copy/move、deleted operations、RAII、return-by-value rules 和 vocabulary types，把部分 convention 提升成 type / object model。
3. 前面的 copy、move、`std::move`、RVO / NRVO 不是零散規則；它們都在回答 object / resource 的語意如何被建立、交付、轉移與結束。

## Next Question

```text
如果 type 承載的不只是 layout，
那一個 type 到底包含哪些東西？

operations？
valid states？
invariants？
cost model？
```

Next:

- [[20-languages/cpp/teaching/Chapter 13 - A Type Is Not Just A Layout]]
