# Learning Path - C++ Object and Resource Semantics

## Purpose

這份 note 是目前正式採用的學習路線草稿。

它不是完整教材，而是用來感受整場教學的「節奏」：

```text
讀者先看到一段自然的 C / C++ code
-> 產生疑問
-> naive model 失敗
-> 才需要下一個 vocabulary
-> vocabulary 反過來解釋剛剛的 code
```

目前採用的風格來自 [[20-languages/cpp/Teaching Notes - C++ Object and Resource Semantics|Teaching Notes]]：

```text
code first
question first
wrong intuition first
terminology later
```

最新判斷是：正式教學不要從 RVO 開場。

RVO 是原始學習動機，但如果主題是「C++ 如何解決 C 的語義保存問題」，更好的入口是 `C Buffer` 的 shallow copy bug。這個例子直接展示：

```text
representation 被 copy 了，
但 ownership semantics 沒有被保留。
```

## Current Thesis

這條路線真正要講的是：

```text
C++ object 不只是 data。

對 non-trivial type 來說，
正確性不只在 bytes / layout，
也在 lifetime、ownership、copy/move operations、valid state、invariants、cost model。
```

更大的 C / C++ 對照是：

```text
C:
    memory + pointer + function + convention

C++:
    object lifetime + type operations + RAII + library vocabulary
```

C 的困境不是做不到，而是很多語義藏在 programmer convention 裡。C++ 的方向不是把 C 包裝得比較好看，而是試圖把 lifetime / ownership / copy / move / valid state 這些語義放進 type operations 和 object model。

## Prologue - I Started From RVO

這段保留原始動機，但不把 RVO 當正式教學入口。

Opening tone:

```text
我原本只是想知道：
為什麼 return by value 不一定 copy？

但這個問題後來一路拉出：
copy / move / ownership / lifetime / RAII / type invariants。
```

RVO 在這條路線裡的角色：

- 原始 hook；
- object delivery 裡的 in-place case；
- 說明「有時候連 move 都不需要」的 return-by-value case study。

RVO 不是：

- 第一個要完整定義的術語；
- 整個 C++ object/resource semantics 的中心；
- RAII / move / ownership 的上位概念。

Reading:

- [[20-languages/cpp/Question Trail - ChatGPT CPP RVO 解釋]]
- [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]]

## Reader Position

這條路線假設讀者：

```text
會寫基本 C / C++
知道 struct / pointer / malloc/free / class / constructor / destructor / STL
聽過 copy / move / RVO 這些詞
但沒有把 lifetime、ownership、value category、RAII、type invariant 串成一個模型
```

所以教學不從 standard wording 開始，也不從「C++17 prvalue」開始。

第一個問題應該非常普通：

```c
typedef struct {
    char* ptr;
    size_t size;
} Buffer;

Buffer a = buffer_create(1024);
Buffer b = a;
```

這段到底 copy 了什麼？

## Big Shape

整條路線的主軸是：

```text
C Buffer shallow copy
-> representation copy vs semantic copy
-> C++ Buffer type operations
-> ownership pressure / Rule of 0/3/5
-> move as ownership transfer
-> std::move / value categories
-> return by value / RVO as no-transfer case
-> noexcept move / generic library contract
-> C convention to C++ type semantics
-> type invariants / regularity / generic programming
```

核心關係：

```text
copy:
    A 和 B 都要一份語義正確的 T。

move:
    A 已經擁有 resource，B 接手 ownership。

RVO / copy elision:
    如果 source object 不需要獨立存在，
    讓 T 直接在 B 的 result storage 形成。
```

一句話定稿：

```text
RVO 告訴我們 object 不一定要被搬。
move 告訴我們 object 背後的 resource 不能只是被 copy。
真正的主題是 C++ 如何用 type operations 保留 ownership、lifetime 與 object semantics。
```

## What This Path Avoids

不要一開始就做這些事：

- 不要一開始定義 RVO。
- 不要一開始丟 `return T{}`。
- 不要一開始丟 `return std::move(x)`。
- 不要一開始講 `prvalue` / `xvalue` / `glvalue` 分類表。
- 不要一開始把 Rust 拉進主線。
- 不要把 factory lambda 放進主線。

這些東西不是不重要，而是它們都需要前置疑問。

## Part 1 - C Buffer: Copy Can Lose Meaning

## Ch1 - C Buffer: Representation Copy Is Not Semantic Copy

## Cold Open

```c
typedef struct {
    char* ptr;
    size_t size;
} Buffer;

Buffer a = buffer_create(1024);
Buffer b = a;
```

## Questions To Ask

```text
這行 `Buffer b = a;` copy 了什麼？
Buffer 能不能 copy？
copy 是 shallow copy 還是 deep copy？
copy 之後誰 owns heap buffer？
誰負責 destroy？
Buffer 的 lifetime 從哪裡開始，到哪裡結束？
```

## Naive Model

```text
struct assignment 就是複製一份 Buffer。
```

## Correct Direction

在 C 這裡，representation 被複製了：

```text
a.ptr == b.ptr
```

但 ownership semantics 沒有被複製：

```text
a.ptr ---+
         +--> same heap buffer
b.ptr ---+
```

如果最後：

```c
buffer_destroy(&a);
buffer_destroy(&b);
```

就可能 double free。

這不是 C 做不到，而是這些規則沒有被 type 本身保留。沒有完整上下文的人，只看 `typedef struct` 和 `Buffer b = a`，無法知道正確語意。

## Main Reading

- [[20-languages/cpp/conversation-notes/Conversation Note - C vs C++ Semantic Lifting and Hidden Semantics]]
- [[20-languages/cpp/deep-dives/Deep Dive - C Convention to Cpp Semantic Lifting]]
- [[20-languages/cpp/object-resource-semantics/Concept - C vs C++ Semantic Lifting]]

## Natural Next Question

```text
如果問題不是「C 能不能做」，
而是語意沒有被 type 保存，
那 C++ 會把這些語意放到哪裡？
```

這自然導向 C++ Buffer。

## Part 2 - C++ Buffer: Meaning Moves Into Type Operations

## Ch2 - Copy / Move / Destroy Become Semantics

## Cold Open

```cpp
class Buffer {
public:
    Buffer(size_t n) : ptr(new char[n]), size(n) {}
    ~Buffer() { delete[] ptr; }

private:
    char* ptr;
    size_t size;
};
```

## Question

```text
如果這個 type 有 destructor，
compiler-generated copy constructor 還對嗎？
```

## Naive Model

```text
class = fields + destructor
copy / move 是 compiler 幫我處理的小事
```

## Correct Direction

如果 type owns resource，copy / move / destroy 就不再是小細節，而是它的核心語義。

Default copy 只會複製 pointer value：

```text
b.ptr = a.ptr
b.size = a.size
```

但這沒有定義 ownership semantics。兩個 object 可能都以為自己 owns 同一塊 memory，最後兩個 destructor 都 delete。

C++ 的方向是讓 type operation 明確回答：

```text
copy:
    deep copy? deleted?

move:
    transfer ownership? moved-from source remains valid?

destructor:
    releases owned resource?
```

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - Buffer Bug RAII Rules and noexcept Move]]
- [[20-languages/cpp/deep-dives/Deep Dive - Object Not Just Data Report Thesis]]
- [[20-languages/cpp/object-resource-semantics/Concept - RAII and Rule of 0 3 5]]

## Natural Next Question

```text
如果 destructor 代表 release resource，
那 constructor / copy / move / assignment 是不是也都必須成為同一組語義？
```

這自然導向 RAII / Rule of 0/3/5。

## Ch3 - Rule Of 0/3/5 Is Ownership Pressure

## Naive Model

```text
Rule of 5 是 C++ 背誦規則。
```

## Correct Direction

Rule of 0/3/5 不是 checklist，而是 ownership pressure：

```text
if destructor frees resource,
then copy/move/assignment must define resource semantics too.
```

Modern direction:

```text
put ownership into RAII handle types
let higher-level domain types follow Rule of Zero
```

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - Buffer Bug RAII Rules and noexcept Move]]
- [[20-languages/cpp/object-resource-semantics/Concept - RAII and Rule of 0 3 5]]

## Natural Next Question

```text
如果 Buffer 不應該 shallow copy，
那它要怎麼被交付、放進 container、或從 function return？
```

這自然導向 move。

## Part 3 - Move: Ownership Transfer When Copy Is Wrong

## Ch4 - If Copy Is Wrong, What Are The Options?

## Question

```text
如果 `Buffer` owns heap memory，
那 `B` 要怎麼拿到一個 `Buffer`？
```

## Naive Model

```text
copy 是唯一自然的 value delivery。
```

## Correct Direction

對 resource-owning type，通常有三種方向：

```text
deep copy:
    語義正確，但可能昂貴。

delete copy:
    安全，但 type 不能用 copy delivery。

move:
    source 不要了，destination 接手 resource ownership。
```

Move 的出現，讓 C++ 可以同時保留：

```text
value-style API
resource ownership
low-cost transfer
```

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - Move Ownership Matrix4x4 and Resource Transfer]]
- [[20-languages/cpp/object-resource-semantics/Concept - Move Semantics and Ownership]]

## Natural Next Question

```text
那 move 到底搬了什麼？
`std::move` 執行完，object 真的被搬走了嗎？
```

這自然導向 `std::move` / value category。

## Ch5 - `std::move` Does Not Move

## Cold Open

```cpp
Buffer b(1024);
std::move(b);
```

## Question

```text
這行執行完，b 被搬走了嗎？
```

## Naive Model

```text
std::move(b)
-> moves b
-> b becomes empty / invalid / moved-from
```

## Correct Direction

```text
std::move(b)
-> static_cast<Buffer&&>(b)
-> produces an xvalue expression
-> later operation may select move constructor / move assignment
```

`std::move` 只是 expression cast / permission signal。真正 transfer resource 的是 type 的 move constructor / move assignment。

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - std move xvalue and Return Path Selection]]
- [[20-languages/cpp/object-resource-semantics/Concept - std move vs Move Constructor]]

## Natural Next Question

```text
如果 `std::move` 只是改變 expression，
那 value category 到底在控制什麼？
```

這自然導向 value categories。

## Ch6 - Value Category Is Not Lifetime

## Why This Chapter Exists

前面已經看到：

```text
b
std::move(b)
Buffer{}
```

它們的差異不能只用「有沒有 object」解釋，還需要 expression category。

## Correct Direction

Expression 有兩個基本面向：

```text
type
value category
```

這章只需要讓讀者抓住：

```text
value category describes expressions;
lifetime describes objects.
```

Examples:

- `b` 是 lvalue expression。
- `std::move(b)` 是 xvalue expression。
- `Buffer{}` 是 prvalue expression。

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - Value Categories Beyond std move]]
- [[20-languages/cpp/object-resource-semantics/Concept - Value Categories - lvalue xvalue prvalue]]
- [Arthur O'Dwyer - Value category is not lifetime](https://quuxplusone.github.io/blog/2019/03/11/value-category-is-not-lifetime/)

## Natural Next Question

```text
move 解決的是 source object 已經存在時的 ownership transfer。
那如果 source object 根本不需要獨立存在呢？
```

這自然導向 return by value / RVO。

## Part 4 - Return By Value And RVO: When Move Is Not Needed

## Ch7 - Return By Value As Object Delivery

## Cold Open

```cpp
Buffer make_buffer() {
    return Buffer(1024);
}
```

## Question

```text
這裡一定要先建立 temporary，再 move 到 caller 嗎？
```

## Naive Model

```text
callee:
    建立 temporary Buffer

return:
    move temporary 到 caller
```

## Correct Direction

`return Buffer(1024)` / `return T{}` 這類 same-type prvalue return 可以直接初始化 result object。

這裡的重點不是「compiler 很聰明幫你省掉 move」，而是：

```text
有些 path 根本不是先有 source object 再 transfer；
語言規則允許 result object 直接被初始化。
```

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - RVO Return Slot and Naive Copy Model]]
- [[20-languages/cpp/object-resource-semantics/Concept - Copy Elision and C++17 prvalue]]

## Natural Next Question

```text
那 named local return 呢？
`return x` 和 `return std::move(x)` 哪個比較好？
```

這自然導向 NRVO / return path selection。

## Ch8 - `return x` vs `return std::move(x)`

## Cold Open

```cpp
Buffer make_buffer() {
    Buffer x(1024);
    return x;
}

Buffer make_buffer_move() {
    Buffer x(1024);
    return std::move(x);
}
```

## Naive Model

```text
copy 很貴。
move 通常比 copy 便宜。
所以 `return std::move(x)` 應該比較保險。
```

## Correct Direction

```text
return x:
    keeps NRVO candidate shape
    fallback may move/copy

return std::move(x):
    changes the return expression to xvalue
    usually loses NRVO candidate shape
```

手動 `std::move` 不是 return local 的保險。

RVO / NRVO 在這條主線中的定位：

```text
move:
    source object 已經存在時，如何低成本轉移 ownership。

RVO / copy elision:
    如果 source object 不必獨立存在，
    就直接在 destination 建構。
```

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - std move xvalue and Return Path Selection]]
- [[20-languages/cpp/object-resource-semantics/Concept - RVO and NRVO]]

## Natural Next Question

```text
如果 move operation 是 type 對 library 的承諾，
那 generic library 怎麼判斷 move path 是否安全？
```

這自然導向 `noexcept move`。

## Part 5 - Generic And Failure Semantics

## Ch9 - `noexcept` Move Is A Promise To Generic Code

## Cold Open

```cpp
std::vector<Buffer> buffers;
buffers.push_back(Buffer(1024));
```

## Question

```text
vector reallocation 時一定會 move elements 嗎？
```

## Naive Model

```text
move 比 copy 快，所以 vector 一定 move。
```

## Correct Direction

Generic containers care about exception guarantee.

`std::move_if_noexcept` roughly encodes:

```text
if nothrow move constructible:
    move
else if not copy constructible:
    move anyway
else:
    copy to preserve strong exception guarantee
```

所以 `noexcept move` 是 type 對 generic code 的承諾。

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - Buffer Bug RAII Rules and noexcept Move]]
- [[20-languages/cpp/object-resource-semantics/Concept - noexcept Move and Container Reallocation]]

## Natural Next Question

```text
我們一路看了 Buffer、copy、move、RVO、RAII、noexcept move。
這些真的只是 C++ 零散規則嗎？
還是它們在解同一個更大的問題？
```

這自然導向 Big Reveal。

## Part 6 - The Big Reveal: From C Convention To C++ Type Semantics

## Ch10 - From C Convention To C++ Semantic Lifting

## Why This Comes Late

這章不適合放開頭。

如果一開始就說：

```text
C++ solves C's semantic gap.
```

讀者可能只會覺得抽象。

但現在讀者已經看過：

- C `Buffer` 展示 representation 可以被複製，但 ownership semantics 留在 convention；
- C++ `Buffer` 展示 copy / move / destroy 必須成為 type operations；
- Rule of 0/3/5 來自 ownership pressure；
- `std::move` 不 move；
- value category 不等於 lifetime；
- move 是 ownership transfer；
- RVO / copy elision 是有些情況下連 move 都不用；
- `noexcept move` 是 type 對 generic library 的承諾。

這時才可以回頭說：

```text
原來這些不是零散規則。
它們都在把 resource semantics 從 convention 提升到 type / object model。
```

## C-Side Questions

```text
這個 pointer 是 owner 還是 view？
誰 free？
copy 是 shallow 還是 deep？
這個 struct 能不能 bitwise copy？
哪個 function 會 invalidate 哪個 resource？
```

## C++-Side Mechanisms

```text
constructor
destructor
copy constructor
move constructor
deleted operations
RAII
type invariant
library vocabulary types
```

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - C Convention to Cpp Semantic Lifting]]
- [[20-languages/cpp/object-resource-semantics/Concept - C vs C++ Semantic Lifting]]
- [[20-languages/cpp/conversation-notes/Conversation Note - C vs C++ Semantic Lifting and Hidden Semantics]]

## Natural Next Question

```text
如果 C++ type 承載了這麼多語義，
那 type 到底是什麼？
```

這自然導向 type invariants。

## Ch11 - A Type Is Not Just A Layout

## Naive Model

```text
type = memory layout + member functions
```

## Correct Direction

```text
Type =
    valid states
  + operations
  + invariants / laws
  + lifetime rules
  + exception safety
  + cost model
```

這裡可以把前面的所有議題收束：

- constructor 建立 valid state；
- destructor 結束 lifetime；
- copy 定義 value/resource duplication；
- move 定義 ownership transfer；
- deleted operation 表示這個 type 不支援某種語義；
- `noexcept` 影響 generic library；
- value-like type 和 resource owner 需要不同 operation set。

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - Type Operations Laws Invariants and Stepanov]]
- [[20-languages/cpp/object-resource-semantics/Concept - C++ Type as Operations and Invariants]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Type Operations Mathematical Analogy and Stepanov]]

## Natural Next Question

```text
如果 type 是 operations + invariants，
generic algorithm 要怎麼知道一個 type 滿足哪些 operations？
```

這自然導向 concepts / regularity。

## Ch12 - Regularity, Concepts, And Generic Programming

## Core Question

```text
algorithm 需要的不是某個 class name，
而是一組 operation requirements。
那 C++ 如何描述這件事？
```

## Correct Direction

Generic programming 不只是 template 能吃任何 type。

```text
algorithm asks for operation requirements
concepts name those requirements
regular / semiregular describe value-like behavior
```

不是每個 type 都該 regular：

```text
unique ownership type:
    copy should not exist

value type:
    copy / equality may be expected semantics
```

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - Type Operations Laws Invariants and Stepanov]]
- [[20-languages/cpp/object-resource-semantics/Concept - C++ Type as Operations and Invariants]]

## End Point

走到這裡，讀者應該能把整條路線收束成：

```text
C++ object/resource semantics is not one feature.
It is a system for making object lifetime, ownership, operations, and invariants explicit enough
for programmers, libraries, and compilers to reason about them.
```

## Extensions

## E1 - Final Storage Beyond Return

Status:

```text
supporting / optional
```

Why it is not main path:

```text
Part 4 已經透過 return by value / RVO 建立 no-transfer case。
如果主線再開一整個 Part 講 final storage，會像重複 RVO。

這段只作為補充：
同一個 final-storage 思想在 container / wrapper / low-level storage 裡如何出現。
```

Topics:

- raw storage is not an object;
- storage vs object lifetime;
- `emplace` is not magic;
- `emplace_back(args...)` vs `emplace_back(T(args...))`;
- `vector` reallocation may still move/copy existing elements;
- placement new / `construct_at` as low-level lifetime control.

Reading:

- [[20-languages/cpp/deep-dives/Deep Dive - Destination First Construction Across RVO Emplace and Placement New]]
- [[20-languages/cpp/deep-dives/Deep Dive - Half RVO Misconception Storage and Lifetime]]
- [[20-languages/cpp/conversation-notes/Conversation Note - RVO Verification Compile Flags and Placement New]]
- [[20-languages/cpp/object-resource-semantics/Concept - Storage vs Object Lifetime]]
- [[20-languages/cpp/object-resource-semantics/Concept - In-place Construction and emplace]]
- [[20-languages/cpp/object-resource-semantics/Concept - Placement New and construct_at]]

## E2 - Factory Lambda And Delayed Construction

Status:

```text
advanced / revisit later
```

Why it is not main path yet:

- It depends on destination-first construction.
- It depends on materialization timing.
- It depends on same-type prvalue direct initialization.
- It often belongs to API boundary design.
- It can blur RVO / emplace / placement new / factory API if introduced too early.

Reading:

- [[20-languages/cpp/deep-dives/Deep Dive - Factory Lambda Is Not Syntax Sugar]]
- [[20-languages/cpp/object-resource-semantics/Concept - Factory Lambda and Delayed Construction]]
- [Arthur O'Dwyer - The Superconstructing Super Elider](https://quuxplusone.github.io/blog/2018/03/29/the-superconstructing-super-elider/)

## E3 - ABI Return Slot

Status:

```text
systems appendix
```

Boundary:

```text
useful implementation model,
not standard source transformation.
```

Reading:

- [[30-systems/abi-function-call/Concept - Function Call Stack and Return Object Storage]]

## E4 - Rust Ownership As Comparison

Status:

```text
oral aside / appendix
```

Role:

```text
C++:
    many ownership/lifetime rules live in RAII, type operations, convention, static analysis

Rust:
    more ownership/borrowing discipline is compiler-enforced
```

Rust is useful as contrast, but should not pull the main C++ teaching path sideways.

Reading:

- [[20-languages/rust/Concept - Rust Ownership Compared With C++ Move]]

## E5 - Other C Semantic Lifting Entry Points

Status:

```text
future expansion
```

Possible openings:

- raw pointer ownership unclear -> `unique_ptr` / `shared_ptr` / `span` / `string_view`;
- out parameter -> return-by-value / `optional` / `expected`;
- lock/unlock -> `lock_guard`;
- enum + union active member -> `variant`;
- `qsort` / `void*` -> templates / iterators / concepts;
- callback + `void* context` -> lambda / callable object / type erasure;
- data race -> mutex / atomic / memory model.

These are useful, but the current teaching route stays centered on Buffer -> move -> RVO.

## E6 - CppCon Watchlist

Status:

```text
further reading
```

Reading:

- [[90-reading-notes/cpp/Reading - CppCon Watchlist for C++ Object Semantics]]
- [CppCon 2019 Back to Basics Track](https://cppcon.org/b2b2019/)

## Chapter Writing Rules

正式寫章節時，每章都應該遵守：

1. Start with code or a concrete failure.
2. Ask the reader what they think happens.
3. State the naive model explicitly.
4. Break the naive model with one sharp example.
5. Only then introduce formal vocabulary.
6. Mark whether a claim is standard semantics, library behavior, guideline, or implementation model.
7. End with exactly three things the reader should now be able to answer.
8. Link back to the relevant Conversation Note.

## Slogan Bank

- `C can do it; the question is where the meaning is stored.`
- `Representation copy is not semantic copy.`
- `Object is not just data.`
- `Rule of 0/3/5 is ownership pressure, not a checklist.`
- `Move is not faster copy.`
- `std::move does not move.`
- `Value category is not lifetime.`
- `Returning by value does not mean copying.`
- `RVO means sometimes there is nothing to move.`
- `noexcept move is a promise to generic code.`
- `A type is not just a layout.`
- `C++ lifts resource conventions into type operations.`

## Next Step

Do not add more topics yet.

Next useful action:

```text
Write Chapter 1 - C Buffer: Representation Copy Is Not Semantic Copy
```

Expected chapter goal:

```text
Use C Buffer shallow copy to make the reader feel:
copying bytes is not enough when a resource has ownership semantics.

Do not mention RVO yet.
Do not mention value categories yet.
Do not turn it into a C vs C++ superiority argument.
```
