# Deep Dive - Half RVO Misconception Storage and Lifetime

## Derived From Conversation Notes

- [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]]
- [[20-languages/cpp/conversation-notes/Conversation Note - RVO Verification Compile Flags and Placement New]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Type Operations Mathematical Analogy and Stepanov]]

## Rebuild Source Rule

這篇從 Conversation Note 中「同一物件生命週期是否能半段 RVO」這條追問重新生成。它不是一般 RVO 教學，而是釐清 source name、storage、object lifetime、ABI return slot 四層。

## Original Questions

- source line `771`: `整個變數不同的生命週期 有的時候是RVO 有的時候是一般`
- source line `1048`: `一個物件生命週期中，一段用 RVO、一段不用 RVO。`
- source line `1187`: `能不能從記憶體角度 畫一張RVO運作解釋圖`
- source line `28606`: `幫我畫示意圖 要符合計算機組織與OS的function call stack、mem對應`
- source line `28624`: `為什麼不在編譯的時候 就直接讓y作用在x的記憶體位置上`

## Conversation Reconstruction

```text
起點：
    你接受 return slot 直覺，但開始把 object lifetime 想成可切換模式。

卡住：
    source variable、physical storage、object lifetime 被混成同一條時間線。

修正：
    RVO/NRVO 不是 object 活到一半後換 storage。

收斂：
    storage decision happens before construction; object lifetime starts in chosen storage.
```

## Why "Half RVO" Is Wrong

錯誤模型：

```text
T obj 在 callee stack 先活著
return 時 obj 搬到 caller
所以 obj 的生命週期前半段一般，後半段 RVO
```

正確模型：

```text
NRVO success:
    result storage is selected first
    obj/result lifetime begins there
    one object

NRVO fail:
    local obj lifetime begins in callee storage
    another object is move/copy constructed in result storage
    two objects
```

所以分析單位不是「同一 object 生命週期的一段」，而是「這個 object lifetime 一開始在哪裡開始」。

> [!note] 圖解定位
> 這張圖對應「同一個 object 不會前半段一般、後半段 RVO」。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_01 (3).png]]

## Four Layers To Keep Separate

```text
source-level name:
    y, x, img 這些程式碼名字

storage:
    一塊符合 size / alignment 的 memory

object lifetime:
    initialization complete 後 typed object 才存在

ABI return slot:
    implementation 用來放 result object 的 storage / hidden parameter
```

對 primitive `int`：

```cpp
int y = 20;
return y;
```

compiler 可能根本不需要真的有 `y` 的 stack slot，可能用 register / constant propagation。這和 non-trivial C++ object 的 lifetime / destructor / copy/move side effects 不同。

## Placement New Is Real, Return Slot Is A Model

Conversation Note 追問：

```cpp
new (return_slot) std::vector<int>();
```

這是真實 placement new 語法，但在 RVO 說明中它是 mental model。

```text
placement new:
    programmer explicitly constructs object in provided storage

RVO return slot:
    language semantics + compiler / ABI implementation

sret / hidden return pointer:
    possible ABI lowering strategy
```

Standard / implementation boundary:

- C++ standard 描述的是 result object 如何由 return operand 初始化，以及哪些 copy/move elision 條件允許把 source/target 視為同一個 object。
- 它不規定 source code 一定會被 lowering 成 `void make(T* return_slot)`。
- hidden return pointer / `sret` 是理解 ABI 和 codegen 的好模型，但不是 portable C++ source semantics。

> [!note] 圖解定位
> placement new / `construct_at` 是指定 storage 開始 object lifetime 的真實工具；RVO return slot 是相鄰的 compiler / ABI mental model。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_21_33 (2).png]]

## External Source Check

- [cppreference - lifetime](https://en.cppreference.com/w/cpp/language/lifetime.html): object lifetime begins after suitable storage and initialization.
- [cppreference - object](https://en.cppreference.com/w/cpp/language/objects): object has type, lifetime, storage duration, alignment, value.
- [cppreference - std::construct_at](https://en.cppreference.com/w/cpp/memory/construct_at): modern library vocabulary for construction at an address.
- [C++ draft - return statement](https://eel.is/c%2B%2Bdraft/stmt.return): return initializes the returned reference or prvalue result object.
- [C++ draft - copy/move elision](https://eel.is/c%2B%2Bdraft/class.copy.elision): elision criteria and source/target same-object treatment.

## Final Mental Model

```text
RVO is not an object moving between stack frames.
RVO is construction placement: choose final/result storage before object lifetime begins.
```

## Source

- [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]]
- [[20-languages/cpp/conversation-notes/Conversation Note - RVO Verification Compile Flags and Placement New]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Type Operations Mathematical Analogy and Stepanov]]
- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `771-1633`, `6042-6253`, `28072-29017`
