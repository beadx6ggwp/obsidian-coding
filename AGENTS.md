# AGENTS.md instructions for C:\Users\david\Documents\obsidian\coding

## 背景

台灣的 CS 畢業生，正備戰資工所（目標：成大 Jserv 研究室、交大、台大圖形或架構相關實驗室）。以逐步撰寫個人專案 Pixel-Renderer（學習 Computer graphics 並從零實作 software renderer）。

## Vault 用途

這是 general coding knowledge base，不只存放 graphics 或 C++，也會包含 fullstack、DevOps、CI/CD、deployment、工具流程、閱讀筆記與專案紀錄。

整理原則：

- 知識按領域放，例如 graphics、systems、web、DevOps。
- 專案按脈絡放，例如 Pixel-Renderer、side project、部署紀錄。
- 尚未整理的內容先放 `00-inbox`，不要因為分類還沒想好而中斷記錄。
- 優先維持可搜尋、可連結、可回顧的 operational notes。

## 技術方向

- 核心興趣：Graphics pipeline、GPU 架構、低層系統程式設計（C++）
- 現在主力：從零用純 C++ 實作 software rasterizer（不依賴 OpenGL/Vulkan/D3D），手動推導每個 GPU pipeline 階段
- 長遠目標：在 FPGA 上重現 NV20 等級的可程式化 GPU pipeline，以軟體渲染器作為 golden model 驗證硬體
- UI 興趣：計畫實作 immediate-mode UI，參考 jserv/libiui 與 microui
- 其他 coding 知識也會整理在此 vault：web fullstack、DevOps、CI/CD、deployment、工具鏈、debugging

## 協作偏好

- 思考方式：First Principles，從最基本事實出發，不接受「大家都這樣做」的答案。
- 語言：繁體中文為主，技術名詞（class 名稱、API、演算法、compiler flags 等）保持 English。
- 學習風格：問題驅動、分層推進（直覺 -> 幾何可視化 -> 數學 -> 程式碼）。
- 解釋概念時，先展示為何 naive 方法失敗，再給解答。

## 溝通注意

- 不喜歡泛泛的建議，希望得到有根據的推理。
- 解釋概念時，可連結到 GPU 架構史（NV10 -> G80 -> RTX）或現代引擎設計（Unreal/Unity RHI）。
- 對 setup、toolchain、debugging 問題，優先給可直接執行的 command 與 side effect 說明。
- 對筆記整理工作，優先保持結構輕量，不要建立過度複雜的分類系統。

