# Vault Map

## Top-Level Folders

| Folder | Purpose |
| --- | --- |
| `00-inbox` | 快速捕捉，還沒分類的筆記、daily notes、臨時想法。 |
| `10-foundations` | DSA、OS、networking、compiler、math、CS core concepts。 |
| `20-languages` | C++、JavaScript、TypeScript、Python、Rust 等語言與 runtime。 |
| `30-systems` | low-level systems、memory、concurrency、debugging、build model。 |
| `40-graphics-gpu` | graphics pipeline、software rasterization、GPU architecture、shader model。 |
| `50-web-fullstack` | frontend、backend、API、database、auth、web app architecture。 |
| `60-devops-infra` | CI/CD、Docker、Linux server、deployment、infra、ops。 |
| `70-tools-workflow` | Git、editor、compiler、debugger、terminal、workflow tools。 |
| `80-projects` | project-specific roadmap、design decisions、debug log、implementation notes。 |
| `90-reading-notes` | books、papers、blogs、source reading。 |
| `99-references` | cheatsheet、glossary、command cookbook、quick lookup。 |
| `_assets` | attachments。由 Obsidian attachment plugin 管理。 |
| `_templates` | note templates。 |
| `_meta` | vault organization、naming rules、writing principles。 |

## Placement Rule

知識本身放在主題資料夾；某個專案如何使用那個知識，放在 `80-projects/<project-name>`。

例：

- `JWT auth` 概念：`50-web-fullstack/auth`
- 某個 side project 的 JWT 實作決策：`80-projects/<project-name>`
- `Dockerfile` 原理：`60-devops-infra/docker-containers`
- Pixel-Renderer 的 build/debug 紀錄：`80-projects/Pixel-Renderer`

