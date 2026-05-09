# Writing Principles

## Preferred Explanation Shape

技術筆記優先使用這個順序：

1. 問題是什麼
2. naive 方法為什麼失敗
3. 直覺模型
4. 幾何 / 系統 / operational view
5. 數學、規格或 API contract
6. implementation
7. verification

## Notes Should Capture

- 這個概念解決什麼問題
- 它依賴哪些前置概念
- 它在哪些情況會失效
- 如何用 command、test、debugger 或小程式驗證
- 它如何連到目前專案或長期方向

## Conversation-Derived Notes

從 ChatGPT 對話、問答紀錄、debug session 或課程討論整理筆記時，不要只保留結論。至少保留一份 `Question Trail`，記錄：

- 當時原始問題的代表性摘錄
- 問題如何從低層現象推進到上位概念
- 哪些疑問造成主題轉向
- 哪些內容被抽成 canonical notes

Canonical note 負責穩定知識；Question Trail 負責保留思考脈絡。

Related method note:

- [[_meta/thinking-methods/Thinking Method - Layered Semantic Decomposition|Thinking Method - Layered Semantic Decomposition]]

For long conversations, use four layers:

- `Question Trail`: why the questions were asked and how the thread shifted.
- `Conversation Note`: one subthread's original question sequence and detailed exploration.
- `Deep Dive`: rewritten reasoning article from naive model to corrected model.
- `Concept`: compressed reusable knowledge anchor.

Do not let `Concept` notes replace `Conversation Note` or `Deep Dive` notes when the value is in the reasoning process.

## Avoid

- 只貼連結，沒有自己的理解
- 只記名詞，沒有 operational model
- 把還沒確認的猜測寫成結論
- 為了分類而延遲記錄
