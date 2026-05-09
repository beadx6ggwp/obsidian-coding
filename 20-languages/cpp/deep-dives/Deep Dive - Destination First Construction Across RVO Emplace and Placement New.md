# Deep Dive - Destination First Construction Across RVO Emplace and Placement New

## Derived From Conversation Notes

- [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]]
- [[20-languages/cpp/conversation-notes/Conversation Note - RVO Verification Compile Flags and Placement New]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Factory Lambda and Delayed Construction]]

## Rebuild Source Rule

這篇從 Conversation Note 裡「跟 RVO 同構的情境有哪些」重新生成。它不是舊的 object delivery 總論，而是專門整理 destination-first construction family。

## Original Questions

- source line `1201`: `那我剛剛提的那種想法 有類似的概念、同構情境嗎`
- source line `1649`: `那跟RVO同構的情境有哪些`
- source line `2203`: `這幾個CASE能不能都劃出解說示意圖`

## Conversation Reconstruction

```text
起點：
    你不是只問 RVO，而是找「不要先做 temporary，再搬到 final place」的相鄰情境。

修正：
    不是所有類比都等價於 RVO。

收斂：
    共同抽象是 destination-first construction。
```

## Same Direction, Not Same Mechanism

類似但不是 RVO：

- `std::vector` reallocation：vector object 不變，但內部 buffer 換位置。
- coroutine frame：local storage lowering，但不是 return object copy elision。
- framebuffer / render target：可類比 final destination，但不是 C++ object lifetime rule。

更接近同構：

- `emplace_back`
- placement new / `std::construct_at`
- `optional<T>::emplace`
- `map::try_emplace`
- uninitialized out storage
- hidden return pointer / `sret`

共同句：

```text
final storage is known
-> do not create T elsewhere
-> begin T lifetime in final storage
```

> [!note] 圖解定位
> `emplace_back` 是最直覺的同構案例：vector 已經知道 element slot，就直接在那裡建構 element。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_21_33 (1).png]]

## Example Ladder

```text
RVO / NRVO:
    function result storage is final destination

emplace_back:
    vector element slot is final destination

optional::emplace:
    optional internal storage is final destination

placement new:
    programmer-provided raw storage is final destination

try_emplace:
    map node's mapped-value storage is final destination
```

## Where The Analogy Stops

`emplace` 不等於「整個操作過程保證沒有 move/copy」。

- `vector.emplace_back(args...)` 可以把新 element 直接建在 element slot。
- 如果 `vector` 需要 reallocate，舊 buffer 裡既有 elements 仍可能 move/copy 到新 buffer。
- `emplace_back(T(args...))` 先 materialize 一個 `T`，再把它交給 `emplace_back`，所以只是在 element slot 裡呼叫 move/copy constructor。
- `optional.emplace(args...)` 沒有 vector reallocation 問題，但如果原本有 value，會先 destroy 舊 contained value，再建新 value。

所以正確說法不是「emplace 保證沒有 move」，而是「在新 object 的 construction site 上，API 有機會直接用 constructor arguments 開始 lifetime」。

> [!note] 圖解定位
> `optional<T>::emplace` 顯示 storage 可以先存在，但 active `T` lifetime 晚一點才開始。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_21_33 (3).png]]

## Why This Became Important Later

這條線直接鋪到 factory lambda：

```text
如果 final storage 很重要，
API 就不該太早要求一個 materialized T。
```

也鋪到 C++ object model：

```text
object 不只是 data fields；
object lifetime 在哪裡開始本身就是語義。
```

## External Source Check

- [cppreference - std::vector::emplace_back](https://en.cppreference.com/w/cpp/container/vector/emplace_back): constructs element in-place.
- [cppreference - std::optional::emplace](https://en.cppreference.com/w/cpp/utility/optional/emplace): constructs contained value in-place.
- [cppreference - std::move_if_noexcept](https://en.cppreference.com/w/cpp/utility/move_if_noexcept): container relocation may choose move or copy based on exception guarantees.
- [cppreference - new expression](https://en.cppreference.com/w/cpp/language/new): placement new.
- [cppreference - std::construct_at](https://en.cppreference.com/w/cpp/memory/construct_at): construct object at address.

## Final Mental Model

```text
RVO is one member of a broader destination-first construction family.
The shared question is: do we already know where T should live?
```

## Source

- [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]]
- [[20-languages/cpp/conversation-notes/Conversation Note - RVO Verification Compile Flags and Placement New]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Factory Lambda and Delayed Construction]]
- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `1201-2967`, `6042-6253`, `3609-5384`
