# Deep Dive - std move xvalue and Return Path Selection

## Derived From Conversation Notes

- [[20-languages/cpp/conversation-notes/Conversation Note - Move Ownership and Value Categories]]

## Rebuild Source Rule

這篇從 `return std::move(img)` 的原始追問生成，聚焦 `std::move`、xvalue、NRVO path selection。move ownership 與完整 value category 另拆成獨立 Deep Dive。

## Original Questions

- source line `6279`: `return std::move(img); // 通常不建議...為什麼是xvalude`
- source line `6601`: `prvalue的p是指什麼`
- source line `11515`: `現在RVO跟MOVE感覺是個緊密關聯的主題 能不能做個完整總整理`

## Conversation Reconstruction

```text
起點：
    你不是只問 return std::move 好不好，而是問為什麼它是 xvalue。

修正：
    std::move 不是 move operation，而是 expression category cast。

收斂：
    return obj 保留 NRVO candidate；return std::move(obj) 改變 return expression path。
```

## `std::move` Does Not Move

```cpp
std::move(img)
```

大致等價於：

```cpp
static_cast<Image&&>(img)
```

它做的是：

```text
lvalue expression
-> xvalue expression
```

它不會：

- 呼叫 move constructor
- 清空 object
- 偷指標
- 釋放 resource

真正的 resource transfer 是被選到的 move constructor / move assignment 做的。

## Return Path Difference

```cpp
T make() {
    T obj;
    return obj;
}
```

```text
NRVO candidate
if NRVO fails, implicit move may still happen
```

```cpp
T make() {
    T obj;
    return std::move(obj);
}
```

```text
return expression is xvalue
not canonical NRVO named-local form
usually blocks NRVO
```

## Calibration: C++23 Does Not Justify `return std::move(local)`

`return obj;` has two important paths:

```text
best case:
    NRVO constructs obj directly in the result object

fallback:
    if elision does not happen, return rules can select move construction
```

Since C++23, a move-eligible return expression is treated as an xvalue for overload resolution when the returned object has to be initialized from it. This improves the fallback path of `return obj;`; it does not make `return std::move(obj);` a better default.

The difference is:

```text
return obj:
    preserves named-local NRVO shape
    then has implicit move/copy fallback

return std::move(obj):
    explicitly makes the operand an xvalue
    usually gives up the NRVO candidate shape
```

> [!note] 圖解定位
> 這張圖把 naive、move fallback、RVO/NRVO 放在一起，能看出 `return std::move(local)` 可能放棄 direct construction。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_47 (3).png]]

## prvalue Is A Different Path

```cpp
return T{};
```

`T{}` 是 prvalue。C++17 後，同型 prvalue return 很多情況直接初始化 result object，不是先產生 temporary 再 move。

所以三種寫法是三種 path：

```text
return T{}:
    prvalue direct initialization

return obj:
    NRVO candidate + move fallback

return std::move(obj):
    explicit xvalue path, usually loses NRVO
```

## External Source Check

- [cppreference - std::move](https://en.cppreference.com/w/cpp/utility/move): `std::move` produces xvalue.
- [cppreference - value categories](https://en.cppreference.com/w/cpp/language/value_category): xvalue / prvalue definitions.
- [cppreference - return statement](https://en.cppreference.com/w/cpp/language/return.html): return and automatic move rules.
- [C++ draft - copy/move elision](https://eel.is/c%2B%2Bdraft/class.copy.elision): NRVO criteria for named automatic local objects.
- [C++ Core Guidelines - F.48](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines?lang=en#Rf-return-move-local): do not return `std::move(local)`.

## Final Mental Model

```text
std::move is a permission signal, not a transport operation.
For returning a local result, return obj usually gives the compiler better paths than return std::move(obj).
```

## Source

- [[20-languages/cpp/conversation-notes/Conversation Note - Move Ownership and Value Categories]]
- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `6279-6731`, `11515-12394`
