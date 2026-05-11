# Chapter 7 - Move Is Ownership Transfer Not Faster Copy

Related:

- [[20-languages/cpp/Learning Path - C++ Object and Resource Semantics]]
- [[20-languages/cpp/teaching/Chapter 6 - Why Move Exists]]
- [[20-languages/cpp/object-resource-semantics/Concept - Move Semantics and Ownership]]
- [[20-languages/cpp/deep-dives/Deep Dive - Move Ownership Matrix4x4 and Resource Transfer]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Move Ownership and Value Categories]]

## Goal

上一章建立了 move 的核心：

```text
copy means duplication.
move means transfer.
```

但這句話還很容易被誤解成：

```text
move = 比 copy 快的版本
```

這章要修正這個直覺。

Move 不是保證更快的 copy。

Move 真正有意義的地方，是 type 背後有某種可以被「接手」的 resource / ownership / state。

## The Naive Model

很多人第一次聽到 move semantics，會形成這個模型：

```text
copy:
    expensive

move:
    cheap
```

這個模型有一部分直覺是對的。

對很多 resource-owning type，move 確實可以便宜很多。

但如果把它說成：

```text
move is faster copy
```

就會漏掉最重要的語意差異。

真正的差別不是：

```text
slow operation vs fast operation
```

而是：

```text
duplication vs transfer
```

## Case 1 - Inline Value Data

先看一個很普通的 value type：

```cpp
struct Matrix4x4 {
    float m[16];
};
```

這個 type 的資料就在 object 裡面。

概念上：

```text
Matrix4x4 object
    contains 16 floats directly
```

如果你 copy：

```cpp
Matrix4x4 b = a;
```

你大概就是複製 16 個 `float`。

那如果你 move 呢？

```text
Matrix4x4 沒有 heap block 可以偷。
沒有 file handle 可以接手。
沒有 pointer ownership 可以轉移。
```

它所有資料都在 object 本體裡。

所以即使你寫出 move constructor，它多半也只能把 16 個 `float` 搬到另一個 object 裡。

也就是：

```text
for inline value data,
move may look almost the same as copy.
```

這不是 compiler 不夠聰明。

而是因為：

```text
沒有外部 resource 可以 transfer。
```

## Case 2 - Resource Owner

現在回到前面的 `Buffer`：

```cpp
class Buffer {
public:
    explicit Buffer(size_t n)
        : ptr(new char[n]), size(n) {}

    ~Buffer() {
        delete[] ptr;
    }

private:
    char* ptr;
    size_t size;
};
```

它的 object 本體裡只有：

```text
ptr
size
```

但真正大的資料在 heap 上：

```text
Buffer object
    ptr ----> heap block
    size
```

如果 deep copy：

```text
a.ptr ----> heap block A
b.ptr ----> heap block B
```

你需要：

```text
allocate new heap block
copy contents
make sure both objects can destroy safely
```

如果 move：

```text
before:
    a.ptr ----> heap block
    b.ptr ----> nothing

after:
    b.ptr ----> heap block
    a.ptr ----> nullptr / empty state
```

這裡 move 可以很便宜，因為它沒有 duplicate heap data。

它只是 transfer ownership。

## What Gets Transferred?

對 `Buffer` 來說，move 不是把所有 bytes 複製一份。

真正被轉移的是：

```text
誰負責那塊 heap memory。
```

也就是：

```text
before move:
    a owns heap block

after move:
    b owns heap block
    a no longer owns heap block
```

這就是 ownership transfer。

所以 move constructor 的核心不只是：

```text
copy pointer value
```

它還要處理 source：

```text
source 不能再以為自己 owns 那塊 memory。
source 之後仍然會被 destructor 處理。
source 必須被放到 safe state。
```

否則只是把前面的 shallow copy bug 換一種方式重演。

## Move Does Not Mean Source Is Destroyed

這點很容易想錯。

Move 之後，source object 還在。

例如概念上：

```cpp
Buffer b = /* move from a */;
```

之後：

```text
b:
    owns the heap block

a:
    still exists
    does not own that heap block anymore
    will still be destructed later
```

所以 move operation 必須留下這個保證：

```text
source object remains safe to destroy.
```

更正式的 `moved-from state` 之後會再講。

現在先用最實際的版本記住：

```text
move 後的 source 不能再 double delete。
```

## When Move Is Not The Right Operation

Move 只有在 source 可以被放棄時才合理。

如果你還需要 `a` 保有原本內容：

```cpp
Buffer a(1024);

Buffer b = /* take ownership from a */;

use(a); // still expecting original buffer
```

那這個設計就有問題。

因為 move 的意思是：

```text
a 放棄原本 resource。
b 接手。
```

如果 `a` 和 `b` 都需要完整資料：

```text
deep copy 才是正確語意。
```

所以不要把 move 當成：

```text
我想要比較快，所以把 copy 都改成 move。
```

比較正確的判斷是：

```text
source 是否還需要原本內容？

如果需要：
    copy / shared ownership / view 等其他設計才可能合理。

如果不需要：
    move 可能合理。
```

## Why The Word "Move" Is Misleading

`move` 這個字容易讓人以為 object 本身被搬走。

但在 C++ 裡，更準確的 mental model 是：

```text
move operation transfers resources/state from one object to another.
```

對 `Buffer`：

```text
resource ownership 被轉移。
```

對 `Matrix4x4`：

```text
沒有 resource ownership 可以轉移，
所以 move 不會神奇地變便宜。
```

這就是為什麼這章標題不是：

```text
Move Is Faster Copy
```

而是：

```text
Move Is Ownership Transfer, Not Faster Copy
```

## Cost Model vs Meaning

Move 常常比較便宜，但這是結果，不是定義。

定義層面：

```text
copy:
    duplicate

move:
    transfer
```

成本層面：

```text
copy of resource owner:
    may allocate and copy resource

move of resource owner:
    may just transfer handle / pointer / ownership

move of inline value type:
    may be similar to copy
```

所以教學上要避免這句：

```text
move is fast copy
```

比較好的句子是：

```text
move can be cheap when the expensive part is a resource handle that can be transferred.
```

## What This Chapter Does Not Explain Yet

這章仍然沒有正式解釋：

```text
std::move
xvalue
rvalue reference
overload resolution
return x vs return std::move(x)
```

因為現在還是在講 operation 的意義。

下一章才會處理 expression 這一層：

```text
如果 move constructor 才真正 transfer resource，
那 std::move(x) 到底做了什麼？
```

## Three Things To Take Away

1. Move 不是 faster copy；copy 是 duplication，move 是 transfer。
2. 對 inline value data，move 可能跟 copy 差不多，因為沒有外部 resource 可以接手。
3. 對 resource owner，move 可以便宜，是因為 destination 接手 resource ownership，而 source 被放到 safe-to-destroy state。

## Next Question

```text
如果 move constructor 才是真正 transfer resource 的 operation，
那 `std::move(x)` 到底做了什麼？
```

Next:

- [[20-languages/cpp/teaching/Chapter 8 - std move Does Not Move]]
