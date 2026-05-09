# Deep Dive - Buffer Bug RAII Rules and noexcept Move

## Derived From Conversation Notes

- [[20-languages/cpp/conversation-notes/Conversation Note - Report Topic Object Delivery RAII and Object Not Data]]
- [[20-languages/cpp/conversation-notes/Conversation Note - C vs C++ Semantic Lifting and Hidden Semantics]]

## Rebuild Source Rule

這篇從 Conversation Note 的 CppCon-style report opening 生成。核心不是背 Rule of 0/3/5，而是用 `Buffer` shallow copy bug 把 RAII、copy/move、noexcept move 串起來。

## Original Questions

- source line `15126`: `Rule of 0 / 3 / 5 是指什麼`
- source line `15650`: `RAII Rule of 5 noexcept move 這些是什麼`
- source line `16846`: 喜歡「你寫 destructor，就一定必須寫 copy constructor」這個角度
- source line `17395`: `從這些之中 我該怎麼更好的切入...像CPP CON那樣 由淺入深`

## Conversation Reconstruction

```text
起點：
    報告需要一個能由淺入深的入口。

找到入口：
    Buffer shallow copy bug 比直接講 RVO 更能引出 C++ object/resource semantics。

收斂：
    RAII、Rule of 3/5/0、noexcept move 都是 resource-owning type 的 lifecycle 壓力。
```

## Buffer Bug

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

如果 compiler 產生 default copy：

```text
b.ptr = a.ptr
b.size = a.size
```

就會出現：

```text
a and b point to same heap buffer
a destructor deletes it
b destructor deletes it again
```

這不是 data layout 問題，而是 operation semantics 問題。

## Rule of 3 / 5 / 0

Rule of 3：

```text
destructor
copy constructor
copy assignment
```

Rule of 5 加上：

```text
move constructor
move assignment
```

Rule of 0 是更現代的方向：

```text
put resource ownership into RAII handle types
let domain objects use default operations
```

## noexcept Move

`vector` reallocation：

```text
allocate new storage
relocate elements
```

如果 move 可能 throw，library 可能選 copy 以維持 exception guarantee。`noexcept move` 是 type 對 library 的承諾：

```text
you can relocate me by move safely
```

## `move_if_noexcept` Makes This Concrete

Standard library code often needs a decision like:

```text
Should I move this element into new storage,
or copy it so the old object remains unchanged if construction fails?
```

`std::move_if_noexcept` encodes that decision:

```text
nothrow move constructible:
    move

copyable but move may throw:
    copy

not copyable:
    move anyway, because there is no copy fallback
```

這讓 `noexcept` move 成為 resource-owning type 的一部分語義：不是因為 move 一定比較快，而是因為 generic containers 需要知道 move path 是否能支撐 exception guarantee。

## External Source Check

- [C++ Core Guidelines - R.1](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines?lang=en#Rr-raii): manage resources with RAII handles.
- [C++ Core Guidelines - C.20/C.21/C.22](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines?lang=en#Rc-zero): Rule of Zero / special member consistency.
- [C++ Core Guidelines - C.66](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines?lang=en#Rc-move-noexcept): move operations should be noexcept.
- [cppreference - std::move_if_noexcept](https://en.cppreference.com/w/cpp/utility/move_if_noexcept): move/copy selection for exception guarantees.

## Final Mental Model

```text
If a type owns a resource, its copy/move/destroy operations are the real semantics.
Rule of 0/3/5 is not a checklist; it is pressure from ownership.
```

## Source

- [[20-languages/cpp/conversation-notes/Conversation Note - Report Topic Object Delivery RAII and Object Not Data]]
- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `15126-18153`
