# AGENTS.md

This file provides guidance to GPT Codex when working with code in this repository.

本檔給未來的 GPT Codex：在這個 workspace 中工作時的整體規範與導航。個別 subproject 的細節不放在這裡，放在各自的 `CLAUDE.md` 或 `README`。

## Workspace layout

此 workspace 是多個獨立 research project 的容器。根目錄本身沒有 build / test / lint。

- **`<subproject>/`** — 每個 subproject 自成一體，擁有自己的 `pyproject.toml`、`.venv`，通常也有自己的 `CLAUDE.md` 或 `README.md`。進入 subproject 工作前，先讀它自己的 doc；不要把其他專案的慣例或本檔未提及的命令套過去。
- **`knowledge/`** — 持久性領域知識的 markdown（見下一節）。
- **`papers/`** — 原始論文 PDF。檔名保留 upstream 原貌（arXiv id 或 conference 提供的檔名），**不要改名**。
- **`models/`**（若存在）— 跨 subproject 共用的模型權重。

## Knowledge base

領域知識、paper 摘要、設計筆記都放在 `knowledge/*.md`。**本檔只是索引，不搬運內容**——編輯原 md，不要 inline 進本檔。

**在寫會碰到下列 topic 的程式碼前，先讀對應 md。**

| Topic | Source of truth |
|---|---|
| Paired model comparison, item-level bootstrap, decoding modes | `knowledge/benchmark_compare.md` |
| *(新 topic 加在這裡，一列一項)* | |

### 新增知識的流程

1. 若有原始 PDF，原檔放入 `papers/`，保留 upstream 檔名。
2. 在 `knowledge/` 新增一份 summary md，檔名用描述性名稱（如 `<topic>_<what_it_documents>.md`）。
3. 若 summary 來自特定 paper，在 summary md 最上方加 `Source: papers/<filename>` 保留連結。
4. 在上表加一列：plain-language topic + 相對路徑，一行之內。
5. 不要把 paper / md 內容 inline 進本檔。

## Environment policy

以下規則對每個 subproject 都適用。

### Python 環境一律用 `uv`

- 新專案：`uv init && uv add <deps>`。
- 既有專案：`uv sync`。
- 不要改用 `pip install`、`conda`、`virtualenv`、`python -m venv`，除非 `uv` 確實無法處理。
- 執行命令用 `uv run ...`，確保進入該 subproject 的 `.venv`。

### 絕對不動系統層安裝

- 不用 `sudo apt`、不用 `pip install --user`、不修改 system Python、不改動 CUDA / NVIDIA driver / 系統 CUDA library。
- 一切都關在 `uv` 建立的 per-project `.venv` 裡。
- 若某任務看起來需要系統層變更，**先停下來問使用者**。CUDA 環境是刻意維穩的，不要冒險。

### GPU

預設 `CUDA_VISIBLE_DEVICES=0`（單卡、id 0）。多卡或其他 device 指派需由使用者在該次任務中明確指定。

### 測試

新專案優先用 `pytest`——主流、CLI 簡單、而且可以直接跑 `unittest`-style 的 `TestCase`，不需改寫。既有專案若已用 `unittest`，維持原樣，不要為了統一而重構。

### 專案級工具由 subproject 自理

formatter、linter、test runner、CLI entrypoint 等命令都屬於各 subproject 自己的 doc，不放本檔。進入 subproject 前若不清楚命令，先讀它的 `CLAUDE.md` 或 `README.md`。

## Working across subprojects

- **不要**把某個 subproject 的具體命令、檔案路徑、架構細節提升到本檔。本檔保持 workspace 抽象層。
- **不要**假設 workspace 根目錄名稱固定（使用者會在不同機器、不同帳號下重用此規範）。
- 跨 subproject 的共用資源只有 `knowledge/`、`papers/`、`models/`；其他各自為政。
