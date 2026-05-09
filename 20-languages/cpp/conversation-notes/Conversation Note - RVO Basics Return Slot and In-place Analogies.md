# Conversation Note - RVO Basics Return Slot and In-place Analogies

## Source Range

- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `9-3608`
- [[00-inbox/ChatGPT-CPP RVO 解釋|Original conversation]]

這段是整串對話的起點：從「RVO 是什麼」一路追到 return slot、NRVO、生命週期誤解、RVO 同構概念、報告方向，以及 C++17 prvalue。

## 1. RVO 是什麼？

Original question:

- source line `9`: `CPP的 return value optimization 是什麼`

討論起點是最基本的 return-by-value：

```cpp
Image makeImage() {
    Image img;
    return img;
}

int main() {
    Image a = makeImage();
}
```

一開始建立的 naive model：

```text
makeImage() 裡建立 img
-> return img 時 copy/move 成 return temporary
-> main 裡再 copy/move 到 a
```

回答展開的核心：

- RVO 是 copy elision 的一種。
- compiler 可以讓 return object 直接建構在 caller 的目的地。
- caller 先準備 return slot，callee 直接 construct。
- 結果可能只剩 constructor + destructor，沒有 copy / move。

當時這不是單純定義，而是在修正「return by value 一定很貴」的直覺。

> [!note] 圖解定位
> 這張圖對應整串對話的第一個 mental model 修正：`return by value` 不必理解成「先在 callee 做物件，再搬兩次」，而可以理解成 caller 先準備 return slot，callee 直接在那個結果位置建構。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_01 (1).png]]

## 2. 你提出的第一個理解：直接分配好位置

Original question:

- source line `273`: `直接在分配的時候就弄好一塊位置 之後操作就直接在這個位置就好`

這個理解其實很接近核心，但需要修正用語。

你當時的想法：

```text
不要產生中間結果。
一開始就弄好一塊位置。
後續操作都直接作用在這塊位置。
```

回答修正：

- 不是「避免掉 C++ 餘留的彈性」。
- 更精準是：compiler 知道 return object 最終要放在哪裡。
- 因此 callee 可以直接在 caller 指定的 result storage 上 construct。

底層 mental model：

```cpp
void make(T* return_slot) {
    new (return_slot) T();
    return_slot->setup();
}
```

這不是標準規定的 source code 轉換，而是幫助理解 hidden return slot / ABI 的模型。

> [!note] 圖解定位
> 這張圖放在這裡，是因為它最貼近你那句「直接在分配的時候就弄好一塊位置」：修正後的說法是，RVO / NRVO 成立時，object 從 construction 開始就可能位於 caller 的 result storage。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_19_44.png]]

## 3. 報告該怎麼講？不是 compiler trick，而是 API design

Original question:

- source line `382`: `這樣報告的時候是要怎麼報阿...有哪些前提...根據原本功能的宏觀角度來看?!`

這裡你開始覺得 RVO 不是單純小技巧，而是要從需求 case 判斷。

回答展開成報告主軸：

1. 問題背景：return by value 看起來很貴。
2. 核心機制：caller 提供 return slot。
3. 哪些 case 適合 return by value？
4. 哪些 case 不適合只靠 RVO？
5. RVO / NRVO / move fallback 差異。
6. 生命週期不是先有兩個 object 再合併。
7. C++17 guaranteed copy elision vs optional NRVO。
8. 實務規則：不要在 local return 寫 `std::move`。

重要的是，這裡已經開始把主題從 `RVO 是什麼` 推到：

```text
如果需求語意是 produce new result，
return by value 可能是乾淨且低成本的 API。
```

> [!note] 圖解定位
> 這張圖對應你把題目從「RVO 是什麼」推到「需求該怎麼判斷」的轉向：先問 final storage、mutation、object lifetime，再決定 return-by-value、emplace、factory 或 mutation API。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_01 (4).png]]

## 4. 生命週期能不能一半 RVO 一半一般？

Original questions:

- source line `771`: `整個變數不同的生命週期 有的時候是RVO 有的時候是一般`
- source line `1034`: `不是同一個物件生命週期中，一段用 RVO、一段不用 RVO。...不同的 return expression`
- source line `1048`: `一個物件生命週期中，一段用 RVO、一段不用 RVO。`

這是這段對話最重要的觀念修正。

你一開始在想：

```text
同一個 object 在生命週期前半段是一般 local，
後半段變成 RVO object？
```

回答修正：

```text
RVO / NRVO 不是 object lifecycle 中途切換模式。
它決定的是 object 從 construction 開始就在哪塊 storage 出生。
```

兩種情況：

```text
NRVO success:
    obj 直接在 caller return slot constructor
    return 時不用 move/copy

NRVO fail:
    obj 在 callee local storage constructor
    return 時 move/copy 到 caller return slot
```

所以分析單位不是「同一 object 的生命週期一段」，而是：

```text
某個 return expression 是否能讓結果 object 從一開始就建在 final storage。
```

> [!note] 圖解定位
> 這張圖對應「同一個 object 不會前半段一般、後半段 RVO」的修正：真正差異是 control flow 能不能讓一個結果物件穩定地對齊 final storage；多個 named local 往往讓 NRVO 變難。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_01 (3).png]]

## 5. 你找相似概念：同構情境

Original questions:

- source line `1201`: `那我剛剛提的那種想法 有類似的概念、同構情境嗎`
- source line `1649`: `那跟RVO同構的情境有哪些`

這裡不是繼續問 RVO，而是在找「一個邏輯值或資源，看起來延續，但底層 storage / representation 改變」的相鄰概念。

回答分兩層：

> [!note] 圖解定位
> 這張圖是同構案例的總覽：真正共同抽象不是「名字都叫 RVO」，而是先決定 final storage，再直接在目的地開始 `T` 的 lifetime。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_01 (5).png]]

### 類似但不是 RVO

- move 後資源延續，但 object lifecycle 不延續。
- `std::string` SSO：同一 string object 內部表示從 inline buffer 變 heap buffer。
- `std::vector` reallocation：vector object 不變，但內部 heap buffer 換位置。
- register allocation / spilling：source-level 變數可能在 register / stack 間變化。
- coroutine frame：local variable 可能被放到 coroutine frame。

> [!note] 圖解定位
> coroutine frame 的圖放在「類似但不是 RVO」，因為它確實有 storage lowering / lifetime placement 的味道，但它不是 return object 的 copy elision。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_21_33 (8).png]]

> [!note] 圖解定位
> framebuffer 類比也只能放在這一類：它能幫助你把 RVO 想成「直接寫 final destination」，但它不是 C++ object lifetime 規則。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_21_33 (9).png]]

### 更接近 RVO 的同構

- `emplace_back`
- placement new / `construct_at`
- `optional<T>::emplace`
- `map::try_emplace`
- `make_unique` / `make_shared`
- uninitialized out storage
- ABI hidden return pointer / `sret`

coroutine frame 和 renderer / framebuffer 類比在前面先放到「類似但不是 RVO」，因為它們可以幫助理解 destination-first，但不是 C++ return object 的 lifetime 同構。

> [!note] 圖解定位
> `emplace_back` 是最直覺的同構案例：vector 已經知道 element slot，就讓 element 直接在那個 slot 建構，而不是先建立 temporary 再 move 進去。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_21_33 (1).png]]

> [!note] 圖解定位
> placement new / `std::construct_at` 是最接近 return slot mental model 的低階形式：storage 先存在，object lifetime 由 explicit construction 在指定位置開始。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_21_33 (2).png]]

> [!note] 圖解定位
> `optional<T>::emplace` 顯示 wrapper 內部可以先有 raw storage，但不一定已經有 active `T` object；等 `emplace` 發生時，`T` 才在 optional 的 internal storage 裡出生。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_21_33 (3).png]]

> [!note] 圖解定位
> `try_emplace` 的重點是 map node 已經是目標 storage，甚至 key 已存在時可以避免建構 mapped value；這比單純「move 很快」更接近 destination-first API design。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_21_33 (4).png]]

> [!note] 圖解定位
> `make_unique` / `make_shared` 的 final storage 是 heap object storage。它不是 return slot，但同樣避免「先做 temporary，再搬到 heap object」這種多餘中間物件。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_21_33 (5).png]]

> [!note] 圖解定位
> out-parameter / uninitialized storage 是把 destination-passing 顯式寫出來。它和 RVO 的差別在於這裡由 programmer 明確傳入 storage，而 RVO 通常由 compiler / ABI 透過 hidden return slot 處理。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_21_33 (6).png]]

> [!note] 圖解定位
> hidden return pointer / `sret` 是 return slot 在 ABI 層的常見實作模型。它應該被理解成 RVO mental model 的底層支撐，而不是另一個 unrelated trick。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_21_33 (7).png]]

這段的真正主題是：

```text
destination-passing / in-place construction:
不要先產生 temporary，再搬到目的地；
而是在目的地開始 lifetime。
```

## 6. 圖與示意圖需求

Original questions:

- source line `1187`: `能不能從記憶體角度 畫一張RVO運作解釋圖`
- source line `2203`: `這幾個CASE能不能都劃出解說示意圖`
- source line `2221`: `給出怎麼分析說哪些需求的case 是可以用這種方式處理...多張示意圖`
- source line `2967`: `有點不太懂 能不能生成各種示意圖`

這些問題不是單純要圖片，而是要把抽象概念落回 memory / object storage。

當時希望看到：

- 沒有 RVO：callee local -> return temp -> caller object。
- RVO / NRVO：caller return slot -> callee direct construct。
- emplace：container slot -> construct element directly。
- placement new：指定 raw storage -> begin object lifetime。
- factory lambda：recipe delayed until final storage known。

這類圖現在已經本地化到 `_assets/0509`，可以直接引用；後續如果要做純文字版，再補 Mermaid / ASCII 作為可 diff 的版本。

## 7. RVO vs NRVO 差異

Original question:

- source line `2981`: `NRVO RVO差異在哪`

整理出的對比：

```text
RVO:
    return expression 是 temporary / prvalue
    例如 return Image{};
    C++17 後很多情況 guaranteed copy elision

NRVO:
    return expression 是 named local variable
    例如 Image img; return img;
    很常見，但不是一般保證
```

記憶體角度：

```text
RVO:
    return expression 本身就是要建構 result object。

NRVO:
    source-level local variable 的 storage 可能直接對齊 caller return slot。
```

> [!note] 圖解定位
> 這張圖對應 RVO / NRVO / move fallback 的差異整理：`return T{}` 是 C++17 後常見 guaranteed copy elision；`return obj` 是 NRVO 機會；控制流程不適合 NRVO 時才 fallback 到 move。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_01 (2).png]]

## 8. C++17 prvalue 進一步升級

在這一段後半，也出現了 C++17 prvalue 的延伸。

重點：

```text
C++17 之後，prvalue 不再必然代表一個 temporary object。
它更像是用來初始化目標 object 的值計算。
```

所以 `return T{};` 不只是 compiler 省掉 move，而是標準語意上很多情況就直接初始化 result object。

這為後面 `factory lambda / delayed construction` 鋪路。

## What Should Be Preserved

這段不該被壓成：

```text
RVO 會省 copy。
```

完整脈絡是：

```text
return by value 看起來很貴
-> caller 可以提供 return slot
-> object 不是中途變 RVO，而是一開始就在 final storage 出生
-> RVO/NRVO 是 object construction placement 問題
-> 同構概念擴展到 emplace、placement new、optional、out storage、renderer target
-> 報告方向從 compiler trick 變成 object delivery / API design
```

## Related Notes

- [[20-languages/cpp/object-resource-semantics/Concept - RVO and NRVO]]
- [[20-languages/cpp/deep-dives/Deep Dive - RVO Return Slot and Naive Copy Model]]
- [[20-languages/cpp/deep-dives/Deep Dive - Half RVO Misconception Storage and Lifetime]]
- [[20-languages/cpp/deep-dives/Deep Dive - Destination First Construction Across RVO Emplace and Placement New]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Factory Lambda and Delayed Construction]]
