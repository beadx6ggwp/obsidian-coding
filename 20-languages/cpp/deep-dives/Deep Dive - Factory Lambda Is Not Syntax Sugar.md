# Deep Dive - Factory Lambda Is Not Syntax Sugar

## Derived From Conversation Notes

- [[20-languages/cpp/conversation-notes/Conversation Note - Factory Lambda and Delayed Construction]]

## Rebuild Source Rule

這篇完全從 factory lambda Conversation Note 生成。核心不是 lambda 語法，而是原始追問：「這是不是語法糖？」

## Original Questions

- source line `3609`: `進階：factory lambda / delayed construction 看不懂 有示意圖嗎`
- source line `3621`: `這一段是語法糖嗎?`
- source line `4000`: `像是unreal、unity 、渲染引擎之類的有真的應用場景嗎`
- source line `4549`: `那瀏覽器呢(chrome) factory lambda / delayed construction的場景嗎`
- source line `4948`: `template<class Factory> T& construct_from(Factory factory) {...}我比較好奇這個的使用場景`

## Conversation Reconstruction

```text
起點：
    factory lambda / delayed construction 看不懂。

卡住：
    T value 版本和 Factory 版本最後都 construct T，看起來只是 lambda 包裝。

修正：
    差異在 T 是否太早 materialize。

收斂：
    final storage 重要、T 不可 move、address-stable object，才是 exact use case。
```

## The Misleading Similarity

```cpp
T& construct_from(T value) {
    return *new (&storage) T(std::move(value));
}
```

```cpp
template<class Factory>
T& construct_from(Factory&& factory) {
    return *::new (&storage) T(std::forward<Factory>(factory)());
}
```

看起來兩者最後都在 `storage` construct `T`。但 `T value` 版本在 function boundary 前已經要求 materialized `T`。

## What Factory Changes

`T value` version:

```text
caller or parameter passing already produced T
final storage becomes known later
T is moved/copied into final storage
```

Factory version:

```text
caller passes recipe
no T exists yet
storage owner invokes recipe when final storage is known
T lifetime begins there
```

## Calibration: Factory Must Preserve Delayed Materialization

factory lambda 不是因為「用了 lambda」所以自動比較強。它只有在 API boundary 真的延後 `T` 的 materialization 時才有意義。

Robust case:

```cpp
construct_from([] {
    return T(args...); // same-type prvalue
});
```

在 C++17 之後，same-type prvalue 可以直接初始化 result object。這是 factory 版本能支撐 immovable `T` 的關鍵。

Fragile case:

```cpp
construct_from([] {
    T local(args...);
    return local; // depends on NRVO plus fallback move/copy
});
```

如果 `T` 是 non-copyable / non-movable，這種 named-local return 不能當作 portable guarantee，因為 NRVO 仍是 permitted optimization，不是 guaranteed copy elision。若 API 內部又先把 factory result 存成 `T value`，那就重新回到 premature materialization。

> [!note] 圖解定位
> 這張圖對應「這不是語法糖」：差異是 materialization 時機與位置。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_16 (4).png]]

## Exact Use Cases

- self-referential or address-sensitive object
- async operation state
- object pool / arena slot
- manual lifetime wrapper
- immovable `T`
- construction must happen inside wrapper / pool / scheduler-owned storage

```text
If moving T after construction changes meaning,
do not ask caller to pass an already-born T.
```

> [!note] 圖解定位
> Address-sensitive operation state 是 exact use case：object 必須直接在 final storage 開始 lifetime。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_37 (4).png]]

## Engine / Browser Examples Must Be Layered

Exact mechanism:

```text
manual storage
emplace / construct_at
operation state
arena / pool allocation
address-stable object
```

Larger analogy:

```text
lazy global
task/callback delayed execution
Blink/Oilpan allocation context
render graph transient resource planning
```

不是所有都叫 factory lambda，但共同壓力是：

```text
do not materialize too early;
wait until destination/lifetime context is known.
```

## External Source Check

- [cppreference - copy elision](https://en.cppreference.com/w/cpp/language/copy_elision.html): same-type prvalue direct initialization context.
- [cppreference - std::optional::emplace](https://en.cppreference.com/w/cpp/utility/optional/emplace): wrapper storage example.
- [cppreference - lifetime](https://en.cppreference.com/w/cpp/language/lifetime.html): storage/lifetime split.
- [CppCon object lifetime class archive](https://cppcon.org/class-2019-obj-lifetime/): follow-up learning.

## Final Mental Model

```text
Factory delayed construction is not lambda magic.
It is an API boundary that avoids requiring T before final storage is known.
```

## Source

- [[20-languages/cpp/conversation-notes/Conversation Note - Factory Lambda and Delayed Construction]]
- [[20-languages/cpp/Image Map - ChatGPT CPP RVO 解釋]]
- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `3609-5384`
