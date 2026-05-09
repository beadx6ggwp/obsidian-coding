# Learning Path - C++ Object and Resource Semantics

## Purpose

這份 note 是目前正式採用的學習路線草稿。

它不是完整教材，而是用來感受整場教學的「節奏」：

```text
讀者先看到一段自然的 C++ code
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

目標不是把所有知識點平均展開，而是讓每個章節都像一場 CppCon-style talk 的下一步：**上一個疑問自然逼出下一個主題**。

## Reader Position

這條路線假設讀者：

```text
會寫基本 C++
知道 class / constructor / destructor / reference / STL
聽過 copy / move / RVO 這些詞
但沒有把 object lifetime、value category、RAII、type invariant 串成一個模型
```

所以教學不從 standard wording 開始，也不從「C++17 prvalue」開始。

第一個問題應該非常普通：

```cpp
T make() {
    T x;
    return x;
}
```

這段到底發生了什麼？

## Core Object Delivery Frame

這條路線真正要問的不是「RVO 這個最佳化是什麼」，而是：

```text
A 要生成一個 T。
B 要怎麼拿到這個 T？
```

C++ 大致有三種語義策略：

```text
1. Copy
   A 有一份，B 也有一份。

2. Move
   A 不要了，由 B 接手 ownership。

3. In-place construction
   不要先在 A 製作再搬到 B；
   直接在 B 的 final storage 製作。
```

所以這份教學的核心句應該是：

```text
RVO 是「避免搬移」。
move 是「搬移不可避免時，便宜地轉移 ownership」。

現代 C++ 的重點不是到處 std::move，
而是讓物件盡量直接出生在它最後要待的位置。
```

這樣介紹 RVO 才不會像是在介紹一個 compiler trick。RVO / copy elision 是 C++ object model 裡「直接在 destination 形成 object」這條設計壓力的代表案例。

## Big Shape

整條路線的主軸是：

```text
return local
-> return path variants
-> std::move misconception
-> value categories
-> move ownership
-> storage / lifetime
-> emplace
-> Buffer bug
-> RAII / Rule of 0/3/5
-> semantic lifting
-> type invariants / generic programming
```

最後的 Big Reveal 是：

```text
C++ 把許多原本在 C 裡靠 convention 維護的 resource semantics
提升到 object lifetime、type operations、RAII、invariants、concepts 裡。
```

但這句話不應該放在第一章講。它應該是讀者經歷完 copy / move / lifetime / RAII 之後，才回頭理解的總結。

## What This Path Avoids

不要一開始就做這些事：

- 不要一開始定義 RVO。
- 不要一開始丟 `return T{}`。
- 不要一開始丟 `return std::move(x)`。
- 不要一開始講 `prvalue` / `xvalue` / `glvalue` 分類表。
- 不要一開始講 C semantic gap。
- 不要把 Rust 拉進主線。
- 不要把 factory lambda 放進主線。

這些東西不是不重要，而是它們都需要前置疑問。

## Prologue - I Only Wanted To Understand RVO

## Why It Exists

這段保留原始動機，不正式教概念。

Opening tone:

```text
我原本只是想知道：
為什麼 return by value 不一定 copy？

但這個問題後來一路拉出：
RVO、move、value categories、storage/lifetime、RAII、type invariants。
```

RVO 在這條路線裡的角色：

- 原始 hook；
- return-by-value 的 case study；
- object delivery 問題的入口。

RVO 不是：

- 第一個要完整定義的術語；
- 整個 C++ object model 的中心；
- RAII / move / lifetime 的上位概念。

Reading:

- [[20-languages/cpp/Question Trail - ChatGPT CPP RVO 解釋]]
- [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]]

## Part 1 - The First Question: Returning A Local

## Ch1 - I Made A Local And Returned It

## Cold Open

```cpp
T make() {
    T x;
    return x;
}
```

這是最自然的起點，因為它符合一般人寫 code 的直覺：

```text
我在 function 裡建立一個東西。
我要把它回傳出去。
```

## Questions To Ask

```text
這段程式會產生幾個 T？
`x` 一定先在 callee stack frame 裡完整存在嗎？
return by value 一定代表 copy 嗎？
caller 拿到的 object 和 local `x` 是什麼關係？
```

## Naive Model

```text
callee:
    建立 local x

return:
    copy/move x 到 caller

caller:
    收到另一個 T
```

這個模型很自然，但它太像 C-style stack frame intuition。它會讓讀者以為「return by value」一定代表先有一個 source object，再搬到 destination。

## What This Chapter Should Teach

這章只建立問題，不急著解完。

核心是讓讀者開始問：

```text
object 是不是一定先在 source 那邊出生？
destination storage 能不能一開始就是 object 的出生地？
```

這就是 object delivery problem 的第一個版本。

它先不回答三分法是哪一個，只要讓讀者意識到：`return x` 不一定等於「A 先完整擁有一個 T，再把另一份 T 交給 B」。

## What Not To Teach Yet

這章先不要完整教：

- `return T{}`
- C++17 prvalue
- xvalue
- `std::move`
- `-fno-elide-constructors`
- ABI hidden return pointer

這些都可以之後再出現。

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - RVO Return Slot and Naive Copy Model]]
- [[20-languages/cpp/object-resource-semantics/Concept - Object Delivery - Copy Move In-place Construction]]

## Conversation Anchor

- [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]]

## Natural Next Question

```text
如果我根本不需要先命名一個 local object，
只是要產生一個 T 當作 return value，
那是不是還要先有 temporary？
```

這自然導向 Ch2 的 `return T{}`。

## Ch2 - What If There Is No Local Name?

## New Code

先保留上一章的版本：

```cpp
T make() {
    T x;
    return x;
}
```

現在加入第二個版本：

```cpp
T make_direct() {
    return T{};
}
```

## Why This Appears Now

這不是突然丟 `T{}`。

它是上一章問題的自然變形：

```text
如果 local name 不是重點，
只是要把一個 T 當成 function result 建出來，
那 object 還需要先出生在別處嗎？
```

## Questions To Ask

```text
`return x` 和 `return T{}` 是同一條 return path 嗎？
`T{}` 是不是先產生 temporary？
`make_direct()` 是否一定需要 copy/move constructor？
為什麼 C++17 後這件事變得更明確？
```

## Naive Model

```text
return T{}:
    先建立一個 temporary T
    再 move/copy 到 caller
```

## Correct Direction

這裡才可以開始命名：

```text
return T{}:
    same-type prvalue can directly initialize the result object

return x:
    named local, NRVO candidate
    if NRVO does not happen, fallback move/copy matters
```

這裡的重點不是「compiler 很聰明幫你省掉 copy」。

更精準的想法是：

```text
有些 path 根本不是先有 temporary 再省掉它；
語言規則允許 result object 直接被初始化。
```

也就是說，`return T{}` 不是單純「compiler 幫你省掉 move」。它是在展示第三種策略：直接在 B 的 result object storage 裡建立 `T`。

## Vocabulary Introduced Here

只在這裡開始引入：

- result object
- copy elision
- prvalue
- RVO
- NRVO
- fallback move/copy

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - RVO Return Slot and Naive Copy Model]]
- [[20-languages/cpp/deep-dives/Deep Dive - Half RVO Misconception Storage and Lifetime]]

## Concept Cards

- [[20-languages/cpp/object-resource-semantics/Concept - Copy Elision and C++17 prvalue]]
- [[20-languages/cpp/object-resource-semantics/Concept - RVO and NRVO]]

## Natural Next Question

```text
如果 `return x` 有 fallback move/copy，
那我怕它 copy 的時候，
是不是可以手動寫 `std::move(x)` 來保險？
```

這自然導向 Ch3。

## Ch3 - The Tempting Insurance: `return std::move(x)`

## New Code

```cpp
T make() {
    T x;
    return std::move(x);
}
```

## Why This Appears Now

這不是建議。

這是一個很自然、但需要被打破的想法：

```text
copy 很貴。
move 通常比 copy 便宜。
如果我怕 `return x` 會 copy，
那我寫 `return std::move(x)` 應該比較保險吧？
```

## Naive Model

```text
std::move tells the compiler:
    please move, do not copy

therefore:
    return std::move(x) should be better than return x
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

所以這章要打掉的是：

```text
手動 std::move 不是 return local 的保險。
```

C++23 改善的是 `return x` 的 fallback path，不是鼓勵你手動 `return std::move(x)`。

這裡要讓讀者抓到：`std::move` 把路徑推向第二種策略，也就是「從已存在的 source object 轉移 ownership」。但如果第三種策略還有機會成立，手動 `std::move` 反而可能把它破壞掉。

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - std move xvalue and Return Path Selection]]

## Concept Cards

- [[20-languages/cpp/object-resource-semantics/Concept - std move vs Move Constructor]]

## Natural Next Question

```text
如果 std::move 不該在這裡當保險，
那 std::move 到底做了什麼？
```

這自然導向 Part 2。

## Part 2 - Moving Without Moving

## Ch4 - `std::move` Does Not Move

## Cold Open

```cpp
T a;
std::move(a);
```

## Question

```text
這行執行完，a 被搬走了嗎？
```

## Naive Model

```text
std::move(a)
-> moves a
-> a becomes empty / invalid / moved-from
```

## Correct Direction

```text
std::move(a)
-> static_cast<T&&>(a)
-> produces an xvalue expression
-> later operation may select move constructor / move assignment
```

`std::move` 只是 expression cast / permission signal。

真正 transfer resource 的是 type 的 move constructor / move assignment。

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - std move xvalue and Return Path Selection]]

## Concept Cards

- [[20-languages/cpp/object-resource-semantics/Concept - std move vs Move Constructor]]

## Natural Next Question

```text
如果 std::move 只是改變 expression，
那 expression category 到底是什麼？
```

這自然導向 value categories。

## Ch5 - Value Category Is Not Lifetime

## Why This Chapter Exists

前面已經看到：

```text
return x
return std::move(x)
return T{}
```

它們的差異不能只用「有沒有 object」解釋，還需要 expression category。

## Naive Model

```text
lvalue = 活得久的東西
rvalue = temporary / 快死掉的東西
xvalue = 已經被 move 的東西
prvalue = temporary object
```

這些說法都容易把 value category 和 object lifetime 混在一起。

## Correct Direction

Expression 有兩個基本面向：

```text
type
value category
```

常用分類：

```text
glvalue = lvalue or xvalue
rvalue  = prvalue or xvalue
```

這章只需要讓讀者抓住：

```text
value category describes expressions;
lifetime describes objects.
```

不要把它變成孤立 taxonomy lecture。所有分類都要回到前面的 code：

- `x` 是 lvalue expression。
- `std::move(x)` 是 xvalue expression。
- `T{}` 是 prvalue expression。

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - Value Categories Beyond std move]]
- [[20-languages/cpp/deep-dives/Deep Dive - std move xvalue and Return Path Selection]]

## Concept Cards

- [[20-languages/cpp/object-resource-semantics/Concept - Value Categories - lvalue xvalue prvalue]]

## Style Reference

- [Arthur O'Dwyer - Value category is not lifetime](https://quuxplusone.github.io/blog/2019/03/11/value-category-is-not-lifetime/)

## Natural Next Question

```text
如果 std::move 只是允許 move operation 被選到，
那 move 真正有意義的情境是什麼？
```

這自然導向 ownership transfer。

## Ch6 - Move Is Not Faster Copy

## Cold Open

```cpp
struct Matrix4x4 {
    float m[16];
};

struct Buffer {
    char* ptr;
    size_t size;
};
```

## Question

```text
這兩個 type 的 move 都一樣有意義嗎？
```

## Naive Model

```text
move = faster copy
```

## Correct Direction

```text
Matrix4x4:
    inline value data
    no resource to steal

Buffer:
    owns heap resource
    move can transfer pointer ownership
```

Move 的核心通常不是 byte-copy speed，而是 ownership transfer。

放回前面的三分法，move 是「A 已經有 object / resource，且 A 願意放棄 ownership」時的交付策略。它不是 modern C++ 的最高目標；更高層的目標仍然是避免不必要的 source object，讓 object 直接在 destination 成形。

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - Move Ownership Matrix4x4 and Resource Transfer]]

## Concept Cards

- [[20-languages/cpp/object-resource-semantics/Concept - Move Semantics and Ownership]]

## Natural Next Question

```text
如果 object 可以在不同地方出生、被 move、被 destroy，
那 memory 和 object lifetime 到底是什麼關係？
```

這自然導向 storage / lifetime。

## Part 3 - Where Objects Are Born

## Ch7 - Raw Storage Is Not An Object

## Cold Open

```cpp
alignas(T) unsigned char storage[sizeof(T)];
```

## Question

```text
這裡已經有一個 T object 嗎？
```

## Naive Model

```text
有一塊足夠大的 memory
-> 那裡就有 object
```

## Correct Direction

C++ 要分開：

```text
storage:
    一塊符合 size / alignment 的 memory

object lifetime:
    initialization 完成後，typed object 才存在
```

這章把前面的 return path 拉回更底層的模型：

```text
重要的不只是 object 被不被 copy；
重要的是 object lifetime 在哪一塊 storage 開始。
```

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - Half RVO Misconception Storage and Lifetime]]
- [[20-languages/cpp/conversation-notes/Conversation Note - RVO Verification Compile Flags and Placement New]]

## Concept Cards

- [[20-languages/cpp/object-resource-semantics/Concept - Storage vs Object Lifetime]]
- [[20-languages/cpp/object-resource-semantics/Concept - Placement New and construct_at]]

## Natural Next Question

```text
如果 object 可以直接在目標 storage 開始 lifetime，
那 container / wrapper 能不能也這樣做？
```

這自然導向 `emplace`。

## Ch8 - `emplace` Is Not Magic

## Cold Open

```cpp
std::vector<T> v;
v.emplace_back(args...);
```

## Question

```text
這是不是保證完全沒有 move？
```

## Naive Model

```text
emplace = no move
```

## Correct Direction

`emplace` 的價值在於：

```text
constructor arguments reach the final construction site
```

這和 RVO / copy elision 呼應的是同一個方向：讓 object 的 lifetime 從 final storage 開始。但 `emplace` 是 library/API 層面的機制，不代表任何情況都沒有 move。

但邊界很重要：

- `emplace_back(args...)` 可以直接建新 element。
- `emplace_back(T(args...))` 已經先 materialized 一個 `T`。
- `vector` reallocation 仍可能 move/copy 既有 elements。
- `optional::emplace` 和 `vector::emplace_back` 的 storage behavior 不同。

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - Destination First Construction Across RVO Emplace and Placement New]]
- [[20-languages/cpp/object-resource-semantics/Concept - In-place Construction and emplace]]

## Style Reference

- [Arthur O'Dwyer - The Superconstructing Super Elider](https://quuxplusone.github.io/blog/2018/03/29/the-superconstructing-super-elider/)

## Natural Next Question

```text
到目前為止我們一直在看 object 怎麼交付、出生、移動。
那如果 object 擁有 resource，copy / move / destroy 會不會變成 type 的核心語義？
```

這自然導向 Buffer bug。

## Part 4 - Resource Semantics

## Ch9 - The Buffer Bug

## Cold Open

```cpp
class Buffer {
    char* ptr;
    size_t size;
public:
    ~Buffer() { delete[] ptr; }
};
```

## Question

```text
compiler-generated copy constructor 對嗎？
```

## Naive Model

```text
class = fields + destructor
copy / move 是 compiler 幫我處理的小事
```

## Correct Direction

如果 type owns resource，copy / move / destroy 就是它的核心語義。

Default copy 只會複製 pointer value：

```text
a.ptr == b.ptr
```

但 ownership 沒有被正確複製，於是可能 double free。

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - Buffer Bug RAII Rules and noexcept Move]]
- [[20-languages/cpp/deep-dives/Deep Dive - Object Not Just Data Report Thesis]]

## Conversation Anchor

- [[20-languages/cpp/conversation-notes/Conversation Note - Report Topic Object Delivery RAII and Object Not Data]]

## Natural Next Question

```text
如果 destructor 代表 release resource，
那 constructor / copy / move / assignment 是不是也都必須成為同一組語義？
```

這自然導向 RAII / Rule of 0/3/5。

## Ch10 - Rule Of 0/3/5 Is Ownership Pressure

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

## Concept Cards

- [[20-languages/cpp/object-resource-semantics/Concept - RAII and Rule of 0 3 5]]

## Natural Next Question

```text
如果 move operation 是 type 對 library 的承諾，
那 library 怎麼判斷 move path 是否安全？
```

這自然導向 `noexcept move`。

## Ch11 - `noexcept` Move Is A Promise To Generic Code

## Cold Open

```cpp
std::vector<Buffer> buffers;
buffers.push_back(Buffer{});
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
我們一路看了 return、move、lifetime、RAII。
這些真的只是 C++ 零散規則嗎？
還是它們在解同一個更大的問題？
```

這自然導向 Big Reveal。

## Part 5 - The Big Reveal

## Ch12 - From C Convention To C++ Semantic Lifting

## Why This Comes Late

這章不適合放開頭。

如果一開始就說：

```text
C++ solves C's semantic gap.
```

讀者可能只會覺得抽象。

但現在讀者已經看過：

- return local object 到底有幾個 lifetime；
- `std::move` 不 move；
- value category 不等於 lifetime；
- move 其實是 ownership transfer；
- raw storage 不等於 object；
- `emplace` 不是 magic；
- shallow copy 會 double free；
- Rule of 0/3/5 來自 ownership pressure。

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
RAII
type invariant
```

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - C Convention to Cpp Semantic Lifting]]
- [[20-languages/cpp/object-resource-semantics/Concept - C vs C++ Semantic Lifting]]

## Conversation Anchor

- [[20-languages/cpp/conversation-notes/Conversation Note - C vs C++ Semantic Lifting and Hidden Semantics]]

## Natural Next Question

```text
如果 C++ type 承載了這麼多語義，
那 type 到底是什麼？
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
- `noexcept` 影響 generic library；
- value-like type 和 resource owner 需要不同 operation set。

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - Type Operations Laws Invariants and Stepanov]]
- [[20-languages/cpp/object-resource-semantics/Concept - C++ Type as Operations and Invariants]]

## Conversation Anchor

- [[20-languages/cpp/conversation-notes/Conversation Note - Type Operations Mathematical Analogy and Stepanov]]

## Natural Next Question

```text
如果 type 是 operations + invariants，
generic algorithm 要怎麼知道一個 type 滿足哪些 operations？
```

這自然導向 concepts / regularity。

## Ch14 - Regularity, Concepts, And Generic Programming

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

## E1 - Factory Lambda And Delayed Construction

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

## E2 - ABI Return Slot

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

## E3 - Rust Ownership As Comparison

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

## E4 - CppCon Watchlist

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
6. Mark whether the claim is standard semantics, library behavior, guideline, or implementation model.
7. End with exactly three things the reader should now be able to answer.
8. Link back to the relevant Conversation Note.

## Slogan Bank

- `Returning by value does not mean copying.`
- `std::move does not move.`
- `Value category is not lifetime.`
- `Move is not faster copy.`
- `Raw storage is not an object.`
- `emplace is not magic.`
- `Rule of 0/3/5 is ownership pressure, not a checklist.`
- `noexcept move is a promise to generic code.`
- `A type is not just a layout.`
- `C++ lifts resource conventions into type operations.`

## Next Step

Do not add more topics yet.

Next useful action:

```text
Write Chapter 1 - I Made A Local And Returned It
```

Expected chapter goal:

```text
Use named-local return to create the object-count question.
Do not fully teach prvalue, xvalue, NRVO, or std::move yet.
Make the reader feel why object delivery is the real problem.
```
