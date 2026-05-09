# Deep Dive - RVO Return Slot and Naive Copy Model

## Derived From Conversation Notes

- [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]]
- [[20-languages/cpp/conversation-notes/Conversation Note - RVO Verification Compile Flags and Placement New]]

## Rebuild Source Rule

這篇從 Conversation Note 的第一段重新生成，不沿用舊 Deep Dive。核心是原始對話如何從「RVO 是什麼」修正 return-by-value 的 naive cost model。

## Original Questions

- source line `9`: `CPP的 return value optimization 是什麼`
- source line `273`: `直接在分配的時候就弄好一塊位置 之後操作就直接在這個位置就好`
- source line `2981`: `NRVO RVO差異在哪`
- source line `5741`: `如果我想編譯看 是要怎麼編譯`

## Conversation Reconstruction

```text
起點：
    return by value 看起來會先建 local，再搬成 return temporary，再搬到 caller。

第一個修正：
    你提出「一開始就弄好位置」，這其實接近 return slot model。

驗證需求：
    你不只想聽概念，也想用 constructor / move / destructor log 編譯觀察。

收斂：
    RVO / NRVO / C++17 prvalue 要分開看。
```

## Naive Model

```cpp
Image makeImage() {
    Image img;
    return img;
}

int main() {
    Image a = makeImage();
}
```

naive execution:

```text
construct img in makeImage
move/copy img to return temporary
move/copy return temporary to a
destroy all intermediate objects
```

這個模型讓 `return by value` 看起來天然昂貴。

## Corrected Model

RVO 的核心不是「搬完再消掉搬運」，而是：

```text
caller needs a result object
-> caller / ABI prepares result storage
-> callee constructs the result object directly there
```

所以你的「直接在分配的時候就弄好一塊位置」是正確直覺，但要補上層級：

```text
source code:
    return img;

language model:
    initialize function result object

implementation model:
    often caller-provided return slot / hidden sret pointer
```

> [!note] 圖解定位
> 這張圖對應整串對話的第一個 mental model 修正：`return by value` 不一定是三段搬運。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_01 (1).png]]

## RVO, NRVO, and C++17 prvalue

```cpp
Image makeA() {
    return Image{};
}

Image makeB() {
    Image img;
    return img;
}
```

`makeA`：

```text
return operand is same-type prvalue
C++17 path is direct initialization of result object
not merely "optimizer removed a temporary"
```

`makeB`：

```text
return operand is named local
NRVO candidate
usually optimized, but not the same guarantee as C++17 prvalue return
```

若 NRVO 失敗，`return img;` 仍可能走 implicit move fallback。這也是後面 `return std::move(img)` 會出問題的背景。

## Verification Layer

Conversation Note 保留了測試方式，這很重要，因為 RVO 容易被講成純信仰。

測試方向：

```bash
g++ -std=c++17 -O0 rvo.cpp -o rvo
g++ -std=c++14 -O0 -fno-elide-constructors rvo.cpp -o rvo14_no_elide
g++ -std=c++17 -O0 -fno-elide-constructors rvo.cpp -o rvo17_no_elide
```

觀察目標：

```text
normal build:
    NRVO may remove copy/move

-fno-elide-constructors:
    shows fallback paths for optional elision

C++17 prvalue:
    some elision is semantic guarantee, not optional optimization
```

## External Source Check

- [C++ draft - class.copy.elision](https://eel.is/c%2B%2Bdraft/class.copy.elision): 校準 copy elision / NRVO 條件。
- [cppreference - copy elision](https://en.cppreference.com/w/cpp/language/copy_elision.html): 校準 C++17 prvalue direct initialization。
- [cppreference - return statement](https://en.cppreference.com/w/cpp/language/return.html): 校準 return object / automatic move。

## Final Mental Model

```text
RVO 不是讓 move 更快。
RVO 是讓 result object 從一開始就可以在結果 storage 裡出生。
```

但 source-backed 筆記要保留精度：

```text
return T{}:
    prvalue direct result initialization

T obj; return obj:
    NRVO candidate

return obj without NRVO:
    possible move fallback
```

## Source

- [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]]
- [[20-languages/cpp/conversation-notes/Conversation Note - RVO Verification Compile Flags and Placement New]]
- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `9-3608`, `5385-6030`

