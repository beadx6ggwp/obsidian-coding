# Learning Path - C++ Object and Resource Semantics

## Purpose

這份 note 是目前正式採用的學習路線草稿。

它不是完整教材，而是用來測試整場教學的「帶入節奏」：

```text
先給一段普通 C++ code
-> 讓讀者真的產生疑問
-> naive model 失敗
-> 才補 vocabulary
-> 最後把 vocabulary 收束成 C++ object/resource semantics
```

目前採用的風格來自 [[20-languages/cpp/Teaching Notes - C++ Object and Resource Semantics|Teaching Notes]]：

```text
code first
question first
wrong intuition first
terminology later
```

## Current Judgment

上一版還是太早攤開：

```text
copy / handoff / in-place
std::move
move
Buffer
RVO
```

這些概念都相關，但如果太早並排，聽眾會不知道自己到底在追哪個問題。

新版改成更單線：

```text
return by value 讓人擔心 copy
-> 先問 copy 到底意味著什麼
-> 再用 Buffer 說明 copy 不只是成本問題
-> copy wrong / expensive / impossible 時，move 才自然出現
-> 最後回頭解釋 RVO / NRVO / return std::move(x)
```

也就是：

```text
不要從 copy / move / in-place 三分法開場。
先從 return-by-value 的 copy 焦慮開始。
再用 Buffer 說明 copy 不是只有成本問題。
最後 move 才自然出現。
```

## Teaching Thesis

這條路線最後要講的是：

```text
C++ object 不只是 data。

一個 object 的建立、交付、轉移、銷毀，
都必須保留 lifetime、ownership、valid state、invariant 和 cost model。
```

但這句不適合當開場。

正式教學應該讓讀者先經歷：

```text
return by value 好像會 copy
-> copy 可能很貴
-> 但更重要的是：copy 必須有語意
-> C Buffer 讓 representation copy 失去 ownership semantics
-> C++ Buffer 把 copy / move / destroy 變成 type operations
-> move 不是 faster copy，而是 source 可被放棄時的 ownership transfer
-> RVO 是另一種更理想的情況：有時候根本沒有 source object 要搬
-> 回頭看，C++ 是在把 C convention 提升成 type semantics
```

## Semantic Axes

RAII、copy / move、RVO 不是同一個概念層級。它們在同一個大框架裡，但分別處理不同軸：

```text
C++ object / resource semantics

Lifetime semantics:
    RAII
    constructor acquires
    destructor releases
    resource lifetime follows object lifetime

Transfer semantics:
    copy
        duplicate / share / view / delete
    move
        ownership transfer
        source remains valid moved-from object

Construction placement semantics:
    RVO / NRVO
    copy elision
    emplace / placement new

Validity / invariant:
    valid state
    moved-from state
    ownership invariant
```

這個分層要在中後段才完整攤開。開場只要讓讀者抓住一件事：

```text
return by value 讓我們不得不問：
caller 那邊到底怎麼出現一個可以正常使用的 T？
```

## Route Shape

新版主線：

```text
return local object
-> why are we worried about copy?
-> copy must produce a usable T, not just similar bytes
-> C Buffer: representation copy can lose meaning
-> C++ Buffer: copy / destroy become type operations
-> Rule of 0/3/5: ownership pressure on special member functions
-> move appears when copy is wrong, expensive, or impossible
-> std::move and value categories explain how move operations are selected
-> return by value revisited: RVO / NRVO / return std::move(x)
-> C convention to C++ type semantics
-> type invariants
```

這版的關鍵調整：

```text
RVO / return by value:
    開場用來產生 copy 焦慮。

Buffer:
    提前到 move 之前。
    用來證明 copy 不是只有效能問題，而是 semantic operation。

Move:
    不再作為早期三分法硬出現。
    它是 copy wrong / expensive / impossible 之後的答案。

C semantic gap:
    不放開頭當抽象 thesis。
    放在最後當 reveal。
```

## Prologue - I Started From RVO

保留原始動機，但不要一開始完整教 RVO。

Opening tone:

```text
我原本只是想知道：
為什麼 return by value 不一定 copy？

但這個問題後來一路拉出：
copy / move / ownership / lifetime / RAII / type invariants。
```

RVO 在這條路線裡的角色：

- 原始 hook；
- return-by-value 的具體疑問；
- 最後用來說明「有時候連 move 都不需要」。

RVO 不是：

- 整場教學的中心；
- 一開始要背的 compiler trick；
- RAII / ownership / move 的上位概念。

Reading:

- [[20-languages/cpp/Question Trail - ChatGPT CPP RVO 解釋]]
- [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]]

## Reader Position

這條路線假設讀者：

```text
會寫基本 C / C++
知道 class / constructor / destructor / reference / STL
聽過 copy / move / RVO 這些詞
但沒有把 lifetime、ownership、value category、RAII、type invariant 串成一個模型
```

所以開場不能是：

- `prvalue` / `xvalue` / `glvalue` 表格；
- C++17 standard wording；
- C vs C++ 大哲學；
- RAII / Rule of Five checklist；
- copy / move / in-place 的完整分類表。

第一個問題要普通到像是初學 C++ 時真的會問：

```text
我在 function 裡建立 local object，然後 return。
caller 那邊到底拿到的是什麼？
```

## Part 1 - Return By Value: Why Are We Worried About Copy?

這一段的任務不是教完 RVO，也不是馬上講 move。

任務只有一個：

```text
讓讀者感受到 return by value 會自然引出 copy 焦慮。
```

## Ch1 - How Many Objects Are In This Code?

Chapter:

- [[20-languages/cpp/teaching/Chapter 1 - How Many Objects Are In This Code]]

## Cold Open

```cpp
T make() {
    T x;
    return x;
}

T y = make();
```

## Questions To Ask

```text
這段 code 裡有幾個 T object？
`x` 和 `y` 是同一個 object 嗎？
return by value 一定 copy 嗎？
local object 會先在 callee 建好，再搬到 caller 嗎？
```

## Naive Model

```text
function 裡有 x。
caller 裡有 y。
所以應該是先 copy 一次。
```

## Correct Direction

先不要急著背 RVO。

這裡真正冒出的問題是：

```text
T 不是單純從 A 的 bytes 複製到 B 的 bytes。
C++ 需要定義「caller 那邊如何出現一個可以正常使用的 T object」。
```

這章只要留下 copy 焦慮：

```text
如果真的 copy，這件事代表什麼？
```

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - RVO Return Slot and Naive Copy Model]]
- [[20-languages/cpp/object-resource-semantics/Concept - RVO and NRVO]]

## Natural Next Question

```text
如果 return by value 讓我們擔心 copy，
那 copy 到底是什麼？
```

這自然導向 copy semantics。

## Ch2 - Copy Is Not Just Moving Bytes

Chapter:

- [[20-languages/cpp/teaching/Chapter 2 - Copy Is Not Just Moving Bytes]]

## Core Question

```text
如果 B 從 A copy 一個 T，
B 得到的是什麼？
```

## Naive Model

```text
copy = 把 A 的 bytes 複製到 B。
```

## Correct Direction

對 trivial data，這個直覺可能暫時夠用。

但對 C++ object 來說，copy 至少應該讓這件事成立：

```text
B 是另一個 T object。
B 可以正常使用。
B 可以正常銷毀。
而且 B 看起來應該代表和 A 相同的內容。
```

也就是說，copy 不能只看 memory 裡的 bits 長得像不像。

比較精準但晚一點才需要的說法是：

```text
copy is a type-defined operation,
not merely byte copying.
```

這一章先不談 move。

只建立一個壓力：

```text
如果 copy 必須產生一個真的能正常使用、正常銷毀的新 object，
那 copy 可能很貴，
也可能根本不是合理操作。
```

## Natural Next Question

```text
什麼情況下 copy 會不是單純 bytes copy？
什麼情況下 copy 甚至會錯？
```

這自然導向 Buffer。

## Part 2 - Copy Is Not Just Cost

這一段把「copy 很貴」修正成更精準的模型：

```text
copy 不只是成本問題。
copy 本身必須有正確語意。
```

Move 不應該在這之前變成主角。

## Ch3 - C Buffer: Representation Copy Is Not Semantic Copy

Chapter:

- [[20-languages/cpp/teaching/Chapter 3 - C Buffer Representation Copy Is Not Semantic Copy]]

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

這不是 C 做不到，而是這些規則沒有被 type 本身保存。沒有完整上下文的人，只看 `typedef struct` 和 `Buffer b = a`，無法知道正確語意。

## Main Reading

- [[20-languages/cpp/conversation-notes/Conversation Note - C vs C++ Semantic Lifting and Hidden Semantics]]
- [[20-languages/cpp/deep-dives/Deep Dive - C Convention to Cpp Semantic Lifting]]
- [[20-languages/cpp/object-resource-semantics/Concept - C vs C++ Semantic Lifting]]

## Natural Next Question

```text
如果 shallow copy 會錯，
那正確 copy 是什麼？
```

這自然導向 deep copy / deleted copy。

## Ch4 - Deep Copy, Deleted Copy, And The Pressure On Type Design

Chapter:

- [[20-languages/cpp/teaching/Chapter 4 - Deep Copy Deleted Copy And Type Design Pressure]]

## Core Question

```text
Buffer 如果要 copy，
合理語意是什麼？
```

## Direction

一個 resource-owning `Buffer` 大概有幾種選擇：

```text
shallow copy:
    cheap, but usually wrong for unique ownership.

deep copy:
    semantically correct if both objects need independent buffers,
    but may be expensive.

deleted copy:
    semantically honest if the type represents unique ownership
    and duplication should not be allowed.
```

這裡才是 move 的前置動機：

```text
如果 deep copy 很貴，
或 copy 根本不符合 type 的語意，
那我們仍然需要某種方式把 object 交出去。
```

但這個方式不能只是「比較快的 copy」。

## Natural Next Question

```text
C++ 能不能讓 type 自己說清楚：
它能不能 copy？
如果能 copy，要怎麼 copy？
如果不能 copy，能不能用另一種方式交付？
```

這自然導向 C++ Buffer。

## Ch5 - C++ Buffer: Copy / Destroy Become Type Semantics

Chapter:

- [[20-languages/cpp/teaching/Chapter 5 - Cpp Buffer Copy Destroy Become Type Semantics]]

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
copy 是 compiler 幫我處理的小事
```

## Correct Direction

如果 type owns resource，copy / destroy 就不再是小細節，而是它的核心語義。

```text
RAII:
    constructor acquire resource
    destructor release resource
    resource lifetime follows object lifetime

copy:
    如果允許 copy，
    必須定義 copy 後兩個 object 各自怎麼使用、怎麼銷毀。
```

C++ 的方向是讓 type operation 明確回答：

```text
copy:
    deep copy? shared ownership? deleted?

destructor:
    releases owned resource?
```

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - Buffer Bug RAII Rules and noexcept Move]]
- [[20-languages/cpp/deep-dives/Deep Dive - Object Not Just Data Report Thesis]]
- [[20-languages/cpp/object-resource-semantics/Concept - RAII and Rule of 0 3 5]]

## Natural Next Question

```text
如果 copy 很貴、copy 不合理、或 copy 被 delete，
那 object 還能怎麼被交付？
```

這自然導向 move。

## Part 3 - If Copy Is Wrong, Move Appears

這一段才正式引入 move。

Move 的動機不是：

```text
現代 C++ 應該到處 std::move。
```

而是：

```text
copy means duplication.

但有些 type：
    copy 很貴；
    copy 不合理；
    copy 根本不可能。

如果 source object 又可以被放棄，
那 transfer 才成為合理設計。
```

## Ch6 - Why Move Exists

Chapter:

- [[20-languages/cpp/teaching/Chapter 6 - Why Move Exists]]

## Core Question

```text
如果 Buffer 不能 shallow copy，
deep copy 又可能很貴，
那我們能不能把 resource 從 A 交給 B？
```

## Correct Direction

Move semantics 表達的是：

```text
source object 不再需要原本 resource 時，
destination 可以接手它的 resource / state。
```

所以：

```text
copy means duplication.
move means transfer.
```

但 move 只有在 source 可以被放棄時才合理。

如果 A 和 B 都需要各自一份完整資料，那 deep copy 才是正確語意。

## Ch7 - Move Is Ownership Transfer, Not Faster Copy

Chapter:

- [[20-languages/cpp/teaching/Chapter 7 - Move Is Ownership Transfer Not Faster Copy]]

## Cold Comparison

```cpp
struct Matrix4x4 {
    float m[16];
};

struct Buffer {
    char* ptr;
    size_t size;
};
```

## Naive Model

```text
move = faster copy
```

## Correct Direction

對 `Matrix4x4` 這種 inline value data，move 可能跟 copy 差不多。

對 `Buffer` 這種 resource owner，move 才有明確語意：

```text
destination 接手 resource ownership。
source 留在 valid moved-from state。
```

所以 move 的重點不是「一定比較快」，而是：

```text
copy 不合理、太貴、或不可能時，
source 可以被放棄，
ownership 可以被轉移。
```

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - Move Ownership Matrix4x4 and Resource Transfer]]
- [[20-languages/cpp/object-resource-semantics/Concept - Move Semantics and Ownership]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Move Ownership and Value Categories]]

## Natural Next Question

```text
那 `std::move(x)` 到底是不是在執行 transfer？
```

這自然導向 `std::move`。

## Ch8 - `std::move` Does Not Move

Chapter:

- [[20-languages/cpp/teaching/Chapter 8 - std move Does Not Move]]

## Cold Open

```cpp
T x;
std::move(x);
```

## Question

```text
這行執行完，x 被搬走了嗎？
```

## Naive Model

```text
std::move(x)
-> moves x
-> x 變成 moved-from
```

## Correct Direction

```text
std::move(x)
-> static_cast<T&&>(x)
-> produces an xvalue expression
-> later operation may select move constructor / move assignment
```

`std::move` 是 expression cast / permission signal。

真正 transfer resource 的是 type 的 move constructor / move assignment。

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - std move xvalue and Return Path Selection]]
- [[20-languages/cpp/object-resource-semantics/Concept - std move vs Move Constructor]]

## Natural Next Question

```text
如果 std::move 只是改變 expression，
那 expression category 到底是什麼？
```

這自然導向 value category。

## Ch9 - Value Category Is Not Lifetime

Chapter:

- [[20-languages/cpp/teaching/Chapter 9 - Value Category Is Not Lifetime]]

## Why This Chapter Exists

前面已經看到：

```text
x
std::move(x)
T{}
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

- `x` 是 lvalue expression。
- `std::move(x)` 是 xvalue expression。
- `T{}` 是 prvalue expression。

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - Value Categories Beyond std move]]
- [[20-languages/cpp/object-resource-semantics/Concept - Value Categories - lvalue xvalue prvalue]]
- [Arthur O'Dwyer - Value category is not lifetime](https://quuxplusone.github.io/blog/2019/03/11/value-category-is-not-lifetime/)

## Natural Next Question

```text
現在我們知道 copy / move 的語意了。
那回到一開始的 return by value，
哪些情況是 copy、move，哪些情況是連 move 都不需要？
```

這自然導向 RVO revisited。

## Part 4 - Return By Value Revisited

這一段回到開頭，但現在讀者已經有足夠 vocabulary。

現在可以精準區分：

```text
copy:
    duplicate a semantic T

move:
    transfer ownership from an existing T

RVO / copy elision:
    initialize result object directly
```

這個三分法現在才適合完整出現，因為 move 已經有情境了。

## Ch10 - `return T{}`: Sometimes There Is Nothing To Move

Chapter:

- [[20-languages/cpp/teaching/Chapter 10 - return T Sometimes There Is Nothing To Move]]

## Cold Open

```cpp
T make() {
    return T{};
}
```

## Syntax Note

這裡的 `T` 是 placeholder，表示某個 type。

`T{}` 是 brace initialization，意思是：

```text
用 empty braces 初始化一個 T。
```

例如：

```cpp
std::string{}
Widget{}
```

這章先假設 `T` 可以這樣初始化。若某個 type 沒有 default constructor，`T{}` 本身就不合法；那是 type construction rule，不是 return-by-value 的問題。

## Question

```text
這裡一定要先建立 temporary，再 move 到 caller 嗎？
```

## Naive Model

```text
callee:
    建立 temporary T

return:
    move temporary 到 caller
```

## Correct Direction

`return T{}` 這類 same-type prvalue return 可以直接初始化 result object。

重點不是「compiler 很聰明幫你省掉 move」，而是：

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

## Ch11 - `return x` vs `return std::move(x)`

Chapter:

- [[20-languages/cpp/teaching/Chapter 11 - return x vs return std move x]]

## Cold Open

```cpp
T make() {
    T x;
    return x;
}

T make_move() {
    T x;
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
現在我們看過 copy、move、destructor、deleted copy、std::move、value category、RVO / NRVO。

這些概念為什麼會被設計出來？

如果 object operation 不承載 ownership / lifetime / copy semantics，
這些語意會在哪裡？
```

這自然導向 Big Reveal：C++ 讓 object operations 承載語意；如果語意沒有被 type / object lifetime 承載，就容易回到 convention。

## Part 5 - The Big Reveal: From C Convention To C++ Type Semantics

這段是昇華主題，不放在開頭。

不是因為一開始要談：

```text
C vs C++
```

而是因為前面已經看過：

- copy 不是 byte movement，而是 type-defined operation。
- deleted copy 表示這個 type 不允許 duplication。
- move 不是 faster copy，而是 ownership transfer。
- destructor 不是普通 function call，而是 object lifetime 結束時的 resource cleanup。
- `std::move` / value category 讓 expression 可以選到正確 operation。
- RVO / NRVO / copy elision 表示有些情況下 transfer 根本不需要。

這時才可以回頭說：

```text
原來這些不是零散規則。
它們都在讓 object operations 承載語意。
```

接著才引入 C-style code 作為對照：

```text
如果 type operation / object lifetime 不承載這些語意，
語意通常不會消失；
它會回到 API convention、comments、documentation、naming、caller discipline。
```

## Ch12 - From C Convention To C++ Semantic Lifting

Chapter:

- [[20-languages/cpp/teaching/Chapter 12 - From C Convention To Cpp Semantic Lifting]]

## Re-reading The Buffer Example

Ch12 不應該突然丟一段新的 C code。

它應該回到 Chapter 3 已經看過的 `Buffer`：

```c
Buffer a = buffer_create(1024);
Buffer b = a;
```

但這次不是重講 double free。

這次的問題是：

```text
`Buffer b = a` 在 representation 層面很清楚：
    copy ptr
    copy size

但 semantic layer 沒有被這個 operation 回答：
    Buffer 能不能 copy？
    copy 是 shallow 還是 deep？
    copy 後誰 owns heap buffer？
    誰 destroy？
```

這樣 C-style code 的出場理由才是：

```text
不是突然講 C。
而是用已知案例展示：
如果 object operation 不承載語意，
語意就會退到 convention / documentation / caller discipline。
```

## C++-Side Mechanisms As Semantic Carriers

```text
constructor:
    建立 valid object / acquire resource

destructor:
    object lifetime 結束時 release resource

copy constructor:
    定義 duplication 的真正語意

move constructor:
    定義 ownership transfer

deleted operations:
    明確表示某種 operation 不符合 type 語意

RVO / copy elision:
    result object 可以直接形成時，不需要 transfer
```

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - C Convention to Cpp Semantic Lifting]]
- [[20-languages/cpp/object-resource-semantics/Concept - C vs C++ Semantic Lifting]]
- [[20-languages/cpp/conversation-notes/Conversation Note - C vs C++ Semantic Lifting and Hidden Semantics]]

## Natural Next Question

```text
如果 C++ 把 hidden convention 提升到 type / object lifetime / type operations，
那我們還能不能把 type 想成「memory layout + functions」？

如果語意被放進 type，
那 type 到底除了 layout 之外還包含什麼？
```

這自然導向 type invariants。

## Ch13 - A Type Is Not Just A Layout

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
- value-like type 和 resource owner 需要不同 operation set。

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - Type Operations Laws Invariants and Stepanov]]
- [[20-languages/cpp/object-resource-semantics/Concept - C++ Type as Operations and Invariants]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Type Operations Mathematical Analogy and Stepanov]]

## End Point

走到這裡，讀者應該能把整條路線收束成：

```text
RVO 告訴我們 object 不一定要被搬。
move 告訴我們 object 背後的 resource 不能只是被 copy。
RAII 告訴我們 resource lifetime 可以跟 object lifetime 綁定。
Rule of 0/3/5 告訴我們 ownership invariant 會壓迫整組 type operations。

真正的主題是：
    C++ 如何用 object lifetime、type operations、library vocabulary
    保留原本容易藏在 C convention 裡的 semantics。
```

## Extensions

這些主題都重要，但不放進主線。它們會把焦點從：

```text
object 如何被建立 / 交付 / 轉移 / 銷毀，
以及這些操作如何保留語意
```

拉到更專門的 library behavior、generic programming、low-level storage、ABI 或 cross-language design。

## E1 - `noexcept` Move And Generic Containers

Status:

```text
extension / library contract case study
```

Why it is not main path:

```text
它不是 object delivery 的核心問題，
而是 generic library 在 exception guarantee 下如何選 copy 或 move。
```

Core question:

```cpp
std::vector<Buffer> buffers;
buffers.push_back(Buffer(1024));
```

```text
vector reallocation 時一定會 move elements 嗎？
```

Correct direction:

```text
move operation 不只要存在，
generic container 還會在意它是否 noexcept。

noexcept move 是 type 對 generic code 的承諾。
```

Reading:

- [[20-languages/cpp/deep-dives/Deep Dive - Buffer Bug RAII Rules and noexcept Move]]
- [[20-languages/cpp/object-resource-semantics/Concept - noexcept Move and Container Reallocation]]

## E2 - Regularity, Concepts, And Generic Programming

Status:

```text
extension / generic programming endpoint
```

Why it is not main path:

```text
主線只需要讓讀者知道 type 不只是 layout。
regularity / concepts 是下一層：
generic algorithm 如何描述它需要哪些 operations。
```

Core question:

```text
algorithm 需要的不是某個 class name，
而是一組 operation requirements。
那 C++ 如何描述這件事？
```

Correct direction:

```text
algorithm asks for operation requirements
concepts name those requirements
regular / semiregular describe value-like behavior
```

Reading:

- [[20-languages/cpp/deep-dives/Deep Dive - Type Operations Laws Invariants and Stepanov]]
- [[20-languages/cpp/object-resource-semantics/Concept - C++ Type as Operations and Invariants]]

## E3 - Final Storage Beyond Return

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

## E4 - Factory Lambda And Delayed Construction

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

## E5 - ABI Return Slot

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

## E6 - Rust Ownership As Comparison

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

## E7 - Other C Semantic Lifting Entry Points

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

These are useful, but the current teaching route stays centered on return-by-value -> copy semantics -> move -> RVO revisited -> type semantics.

## E8 - CppCon Watchlist

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

- `Returning by value does not mean copying.`
- `Copy must produce a usable object, not just similar bytes.`
- `Deep copy can be correct and expensive.`
- `Deleted copy can be the honest type operation.`
- `Move appears when copy is wrong, expensive, or impossible and the source can be abandoned.`
- `std::move does not move.`
- `Value category is not lifetime.`
- `Move is not faster copy.`
- `Representation copy is not semantic copy.`
- `Object is not just data.`
- `Rule of 0/3/5 is ownership pressure, not a checklist.`
- `RVO means sometimes there is nothing to move.`
- `A type is not just a layout.`
- `C can do it; the question is where the meaning is stored.`
- `C++ lifts resource conventions into type operations.`

## Next Step

Do not add more topics yet.

Next useful action:

```text
Write Chapter 13 - A Type Is Not Just A Layout
```

Expected chapter goal:

```text
Close the main teaching arc.

Use Chapter 12's semantic lifting conclusion as the bridge.

Explain that a C++ type is not just memory layout plus functions.
It also encodes valid states, operations, invariants, lifetime rules, and cost model.
```
