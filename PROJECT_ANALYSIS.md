# Claude Code 專案完整分析報告

> **生成日期**：2026-01-15
> **分析方法**：深度專案探索與架構分析
> **版本**：v2.0.0（整合版）

---

## 目錄

1. [專案概覽](#1-專案概覽)
2. [架構設計](#2-架構設計)
3. [核心功能模組](#3-核心功能模組)
4. [Hook 系統深度分析](#4-hook-系統深度分析)
5. [插件生態系統](#5-插件生態系統)
6. [MCP 整合系統](#6-mcp-整合系統)
7. [安全機制](#7-安全機制)
8. [GitHub 自動化工作流](#8-github-自動化工作流)
9. [閉源接口協議](#9-閉源接口協議)
10. [配置管理系統](#10-配置管理系統)
11. [開發指南](#11-開發指南)
12. [關鍵文件索引](#12-關鍵文件索引)
13. [架構優缺點分析](#13-架構優缺點分析)

---

## 1. 專案概覽

**Claude Code** 是 Anthropic 官方的 CLI AI 編程助手，採用**開源插件 + 閉源核心**的架構設計。本 repo 是其插件生態系統，核心 CLI 引擎位於閉源的 `@anthropic-ai/claude-code` npm 包中。

### 1.1 核心統計數據

| 指標 | 數值 |
|------|------|
| 官方插件 | **13 個** |
| 斜線命令 | **40+** |
| 專業 Agent | **19+** |
| 技能模組 | **10+** |
| Hook 事件類型 | **9 個** |
| GitHub Actions 工作流 | **11 個** |
| Markdown 文檔 | **92+ 個** |
| Python 核心代碼 | **~1,100+ 行** |
| TypeScript 腳本 | **~500+ 行** |

### 1.2 技術棧

| 技術 | 版本/說明 | 用途 |
|------|---------|------|
| **Node.js** | 20+ | 運行時環境 |
| **Python** | 3.7+ | Hook 系統和規則引擎（零外部依賴） |
| **TypeScript** | latest | GitHub 自動化腳本 |
| **Bash/Shell** | - | 環境配置和開發腳本 |
| **Markdown + YAML** | Frontmatter | 聲明式配置 |

### 1.3 官方插件清單

| 插件名稱 | 類型 | 功能 | 關鍵特性 |
|---------|------|------|---------|
| **feature-dev** | 開發工具 | 7 階段功能開發工作流 | 3 個協調 Agent |
| **code-review** | 開發工具 | PR 自動化代碼審查 | 4 個並行 Agent + 信心度評分 |
| **pr-review-toolkit** | 開發工具 | 專門評審 Agent 集合 | 6 個獨立 Agent |
| **agent-sdk-dev** | 開發工具 | Agent SDK 應用開發 | 自動化驗證 |
| **plugin-dev** | 開發工具 | 插件開發工具包 | 7 個專門 Skill |
| **hookify** | 生產力 | 可配置規則鉤子系統 | 規則引擎 + 動態載入 |
| **commit-commands** | 生產力 | Git 工作流自動化 | commit / push / PR |
| **security-guidance** | 生產力 | 安全性提示鉤子 | 9 種危險模式檢測 |
| **ralph-wiggum** | 迭代工具 | 迭代自參考循環 | Stop hook 自反饋 |
| **explanatory-output-style** | 學習風格 | 教學性輸出風格 | SessionStart Hook |
| **learning-output-style** | 學習風格 | 學習模式輸出 | 互動式決策點 |
| **frontend-design** | 學習風格 | 前端設計工具 | 高質量 UI 代碼生成 |
| **claude-opus-4-5-migration** | 遷移指南 | 模型遷移指南 | Sonnet → Opus 升級 |

---

## 2. 架構設計

### 2.1 整體架構圖

```
┌─────────────────────────────────────────────────────────────────┐
│            @anthropic-ai/claude-code (閉源核心)                   │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ • Plugin Loader (自動發現機制)                              ││
│  │ • Hook Dispatcher (stdin/stdout 通信)                       ││
│  │ • MCP Manager (外部工具集成)                                ││
│  │ • Frontmatter Parser (配置解析)                             ││
│  │ • Environment Injector (變量注入)                           ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
                        ↓ ↑ JSON 協議 / Markdown 配置
┌─────────────────────────────────────────────────────────────────┐
│              本開源倉庫（插件生態系統）                            │
│  • /plugins (13 個官方插件)                                      │
│  • .claude/commands/ (專案級命令)                                │
│  • .github/workflows/ (CI/CD 工作流)                            │
│  • scripts/ (自動化腳本)                                         │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 目錄結構

```
claude-code/
├── .claude/                          # 專案級配置和命令
│   ├── commands/                     # 專案特定的 slash 命令
│   │   ├── commit-push-pr.md
│   │   ├── dedupe.md
│   │   └── oncall-triage.md
│   └── settings.json
├── .claude-plugin/                   # 插件市場配置
│   └── marketplace.json              # 13 個插件清單
├── plugins/                          # 13 個官方插件（1.2 MB）
│   ├── agent-sdk-dev/
│   ├── code-review/
│   ├── feature-dev/
│   ├── hookify/                      # 核心規則引擎
│   ├── plugin-dev/
│   ├── pr-review-toolkit/
│   ├── security-guidance/
│   ├── commit-commands/
│   ├── ralph-wiggum/
│   ├── explanatory-output-style/
│   ├── learning-output-style/
│   ├── frontend-design/
│   └── claude-opus-4-5-migration/
├── scripts/                          # 自動化腳本
│   ├── auto-close-duplicates.ts
│   └── backfill-duplicate-comments.ts
├── examples/                         # 參考示例
│   └── hooks/
├── .github/                          # GitHub 工作流配置
│   ├── workflows/                    # 11 個 CI/CD 工作流
│   └── ISSUE_TEMPLATE/
└── .devcontainer/                    # 開發環境配置
    ├── devcontainer.json
    └── Dockerfile
```

### 2.3 插件內部結構規範

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json              # 插件元數據（必需）
├── commands/                    # 斜線命令 (.md)
├── agents/                      # AI 代理定義 (.md)
├── skills/                      # 知識模塊
│   └── skill-name/
│       ├── SKILL.md             # 核心文檔
│       ├── references/          # 詳細指南
│       ├── examples/            # 工作示例
│       └── scripts/             # 驗證工具
├── hooks/                       # 事件處理器
│   ├── hooks.json
│   └── *.py
├── .mcp.json                    # MCP 服務器配置
└── README.md
```

### 2.4 核心設計模式

| 模式 | 實現位置 | 作用 |
|------|---------|------|
| **觀察者模式** | Hook 系統（9 個事件點） | 事件驅動，細粒度控制 |
| **策略模式** | 模型選擇（haiku/sonnet/opus） | 根據任務複雜度選擇模型 |
| **工廠模式** | 標準目錄結構 + 元數據 | 自動加載插件組件 |
| **規則引擎模式** | Hookify 插件 | 靈活的行為控制 |
| **命令模式** | Markdown 命令定義 | 聲明式命令註冊 |

---

## 3. 核心功能模組

### 3.1 Tool 系統

**工具授權機制（Frontmatter）：**
```yaml
---
allowed-tools: Read, Grep, Bash(git:*, npm:*)
model: sonnet
---
```

**可用工具列表：**

| 類別 | 工具 |
|------|------|
| 基礎 | `Bash`, `Edit`, `Write`, `Glob`, `Grep`, `Read`, `LS` |
| 高級 | `TodoWrite`, `WebFetch`, `WebSearch`, `NotebookRead` |
| MCP | `mcp__plugin_<name>_<server>__<tool>` |

### 3.2 Agent 系統

**Agent 定義格式：**
```markdown
---
name: code-reviewer
description: Use this agent for code review requests
model: sonnet
color: blue
tools: ["Read", "Grep"]
---

You are an expert code reviewer...
```

**模型選擇策略：**

| 模型 | 用途 | 場景 |
|------|------|------|
| `haiku` | 快速、輕量 | 簡單檢查、快速驗證 |
| `sonnet` | 平衡性能 | 推薦默認、一般任務 |
| `opus` | 最大能力 | 複雜分析、關鍵決策 |

**官方 Agent 統計：**

| 插件 | Agent | 用途 |
|------|-------|------|
| feature-dev | code-explorer, code-architect, code-reviewer | 功能開發工作流 |
| code-review | 4+ 個並行 Agent | 多方面代碼審查 |
| pr-review-toolkit | 6 個專業 Agent | PR 深度評審 |
| agent-sdk-dev | 2 個驗證 Agent | SDK 應用校驗 |

### 3.3 Skill 系統

**漸進式披露設計：**

```
1. 元數據（始終加載）
   ├─ 名稱 + 描述 (~100 字)
   └─ 位置信息

2. SKILL.md 正文（觸發時加載）
   ├─ 核心指導 (<5k 字)
   └─ 最佳實踐

3. 資源（按需加載）
   ├─ references/   （詳細指南）
   ├─ examples/     （工作示例）
   └─ scripts/      （驗證工具）
```

**官方 Skill：**

| Skill | 位置 | 功能 |
|-------|------|------|
| hook-development | plugin-dev | Hook 編寫指南 |
| command-development | plugin-dev | 命令開發工具 |
| agent-development | plugin-dev | Agent 開發指南 |
| mcp-integration | plugin-dev | MCP 集成教程 |
| plugin-structure | plugin-dev | 插件結構規範 |
| frontend-design | frontend-design | UI 設計工具 |
| claude-opus-4-5-migration | claude-opus-4-5-migration | 模型遷移 |

---

## 4. Hook 系統深度分析

### 4.1 支持的事件類型

| 事件 | 觸發時機 | 主要用途 | 示例插件 |
|------|---------|---------|---------|
| `PreToolUse` | 工具執行前 | 驗證、權限檢查、攔截 | Hookify, security-guidance |
| `PostToolUse` | 工具執行後 | 結果處理、日誌記錄 | 性能監控 |
| `Stop` | Agent 停止前 | 完成度檢查 | Ralph-Wiggum |
| `SubagentStop` | 子代理停止前 | 子代理驗證 | - |
| `SessionStart` | 會話開始 | 加載上下文 | explanatory-output-style |
| `SessionEnd` | 會話結束 | 清理資源 | - |
| `UserPromptSubmit` | 用戶提交提示 | 輸入驗證 | Hookify 消息提醒 |
| `PreCompact` | 上下文壓縮前 | 敏感信息處理 | 隱私保護 |
| `Notification` | 發送通知時 | 監控記錄 | - |

### 4.2 Hook 配置格式

**文件位置：** `plugins/*/hooks/hooks.json`

```json
{
  "description": "Hook description",
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "python3 ${CLAUDE_PLUGIN_ROOT}/hooks/pretooluse.py",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

### 4.3 stdin/stdout 通信協議

**輸入格式（stdin）— 閉源核心發送：**
```json
{
  "hook_event_name": "PreToolUse",
  "session_id": "abc-123",
  "cwd": "/project/root",
  "tool_name": "Bash|Edit|Write",
  "tool_input": {
    "command": "rm -rf /tmp/test"
  },
  "transcript_path": "/tmp/transcript.txt"
}
```

**輸出格式（stdout）— Hook 腳本返回：**
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow|deny|ask",
    "updatedInput": {}
  },
  "systemMessage": "警告或錯誤信息"
}
```

**Exit 碼語義：**

| 退出碼 | 含義 | 行為 |
|--------|------|------|
| `0` | 成功/允許 | 操作繼續 |
| `2` | 阻塞 | stderr 回饋給 Claude |
| 其他 | 非阻塞錯誤 | 記錄錯誤但繼續 |

### 4.4 Hookify 規則引擎實現

**文件位置：** `plugins/hookify/core/`

**核心類：**

```python
# rule_engine.py (314 行)
class RuleEngine:
    def evaluate_rules(rules: List[Rule], input_data: Dict) -> Dict:
        # 1. 檢查規則是否匹配
        # 2. 優先級：block > warn > allow
        # 3. 返回 JSON 決策

    def _rule_matches(rule: Rule, input_data: Dict) -> bool:
        # 工具匹配 + 條件匹配（AND 邏輯）

    def _check_condition(condition: Condition, ...) -> bool:
        # 支持 6 種操作符

# config_loader.py (298 行)
class ConfigLoader:
    def load_rules(event: Optional[str]) -> List[Rule]:
        # 掃描 .claude/hookify.*.local.md
        # 解析 YAML Frontmatter + Markdown 正文
```

**條件操作符：**

| 操作符 | 語義 | 用途 |
|--------|------|------|
| `regex_match` | 正則表達式匹配 | 複雜模式 |
| `contains` | 字符串包含 | 簡單檢查 |
| `not_contains` | 字符串不包含 | 排除模式 |
| `equals` | 精確相等 | 完全匹配 |
| `starts_with` | 開頭匹配 | 前綴檢查 |
| `ends_with` | 結尾匹配 | 擴展名檢查 |

**性能優化：**

```python
@lru_cache(maxsize=128)
def compile_regex(pattern: str) -> re.Pattern:
    return re.compile(pattern, re.IGNORECASE)
```

- 首次編譯：~7ms
- 緩存命中：~0.1ms（**70x 加速**）

### 4.5 規則配置示例

**文件位置：** `.claude/hookify.*.local.md`

```markdown
---
name: warn-dangerous-rm
enabled: true
event: bash
pattern: rm\s+-rf
action: warn
conditions:
  - field: command
    operator: regex_match
    pattern: rm\s+-rf
---

⚠️ Dangerous rm command detected!
```

---

## 5. 插件生態系統

### 5.1 關鍵插件深度分析

#### Hookify 插件 — 核心架構

```
plugins/hookify/
├── .claude-plugin/plugin.json
├── commands/
│   ├── hookify.md              # 主命令：分析並創建規則
│   ├── list.md                 # 列出所有規則
│   └── help.md                 # 幫助
├── agents/
│   └── conversation-analyzer.md # 分析對話中的問題行為
├── skills/
│   └── writing-rules/SKILL.md   # 規則編寫指南
├── hooks/
│   ├── hooks.json              # Hook 配置
│   ├── pretooluse.py           # PreToolUse 執行器
│   ├── posttooluse.py          # PostToolUse 執行器
│   └── stop.py                 # Stop 事件處理
├── core/
│   ├── rule_engine.py          # 規則評估引擎（314 行）
│   ├── config_loader.py        # 配置加載器（298 行）
│   └── __init__.py
├── matchers/ 和 utils/
└── README.md
```

**工作流程：**
```
/hookify "Block dangerous rm commands"
    ↓
conversation-analyzer Agent 分析對話
    ↓
生成 .claude/hookify.*.local.md
    ↓
下次工具執行時 pretooluse.py 觸發
    ↓
rule_engine.py 評估規則
    ↓
返回 JSON 決策（allow/deny/ask）
```

#### Feature-Dev 插件 — 7 階段工作流

```
plugins/feature-dev/
├── commands/
│   └── feature-dev.md          # /feature-dev 主命令
├── agents/
│   ├── code-explorer.md        # 黃色 - 深度代碼分析
│   ├── code-architect.md       # 綠色 - 架構設計
│   └── code-reviewer.md        # 紅色 - 質量審查
└── skills/ (如有)
```

**工作流階段：**
1. 需求分析與代碼探索 → code-explorer
2. 架構設計與實現規劃 → code-architect
3. 功能實現
4. 單元測試編寫
5. 集成測試
6. 代碼審查 → code-reviewer
7. 性能優化與文檔

#### Plugin-Dev 插件 — 開發工具包

```
plugins/plugin-dev/
├── commands/
│   └── create-plugin.md        # 8 階段引導式插件創建
├── agents/
│   ├── agent-creator.md
│   ├── plugin-validator.md
│   └── skill-reviewer.md
└── skills/
    ├── hook-development/       # Hook 編寫指南
    ├── mcp-integration/        # MCP 集成教程
    ├── command-development/    # 命令開發
    ├── agent-development/      # Agent 編寫
    ├── plugin-structure/       # 結構規範
    ├── plugin-settings/        # 配置管理
    └── skill-development/      # Skill 編寫
```

### 5.2 插件間依賴關係

```
核心基礎：
  ├─ hookify（規則引擎）
  └─ security-guidance（安全警告）

功能開發鏈：
  ├─ feature-dev（7 階段工作流）
  ├─ code-review（自動審查）
  └─ pr-review-toolkit（深度評審）

工具鏈：
  ├─ commit-commands（Git 自動化）
  ├─ agent-sdk-dev（SDK 開發）
  └─ plugin-dev（插件開發）

學習和優化：
  ├─ explanatory-output-style（教學）
  ├─ learning-output-style（交互學習）
  └─ frontend-design（UI 設計）

特殊用途：
  ├─ ralph-wiggum（自迭代）
  └─ claude-opus-4-5-migration（模型升級）
```

---

## 6. MCP 整合系統

### 6.1 服務器類型

| 類型 | 傳輸方式 | 狀態 | 用途 | 配置示例 |
|------|---------|------|------|---------|
| **stdio** | 進程 stdin/stdout | 有狀態 | 本地工具 | `"command": "npx"` |
| **SSE** | HTTP Server-Sent Events | 有狀態 | 雲服務 | `"type": "sse"` |
| **HTTP** | REST API | 無狀態 | REST 服務 | `"type": "http"` |
| **WebSocket** | 雙向實時 | 有狀態 | 實時通信 | `"type": "ws"` |

### 6.2 配置格式

**文件位置：** `plugins/*/.mcp.json`

**stdio（本地進程）：**
```json
{
  "database": {
    "command": "${CLAUDE_PLUGIN_ROOT}/servers/db-server.js",
    "args": ["--config", "config.json"],
    "env": {
      "DATABASE_URL": "${DATABASE_URL}"
    }
  }
}
```

**SSE（雲服務）：**
```json
{
  "asana": {
    "type": "sse",
    "url": "https://mcp.asana.com/sse"
  }
}
```

**HTTP（REST API）：**
```json
{
  "api-server": {
    "type": "http",
    "url": "https://api.example.com/mcp",
    "headers": {
      "Authorization": "Bearer ${API_TOKEN}"
    }
  }
}
```

### 6.3 工具命名約定

```
mcp__plugin_<plugin-name>_<server-name>__<tool-name>

示例：
- mcp__plugin_asana_asana__asana_create_task
- mcp__plugin_github_github__search_issues
- mcp__plugin_myplug_database__query
```

### 6.4 認證機制

| 方式 | 配置 | 用途 |
|------|------|------|
| **OAuth 2.0** | 自動處理 | SSE 服務（如 Asana） |
| **Bearer Token** | `headers.Authorization` | HTTP/WebSocket |
| **API Key** | `headers.X-API-Key` | 自定義服務 |
| **環境變量** | `env` 字段 | stdio 服務器 |

---

## 7. 安全機制

### 7.1 權限控制系統

**allowed-tools 格式：**
```yaml
allowed-tools:
  - Read
  - Bash(git:*, npm:*)        # 正則過濾
  - Write                     # 完全訪問
  - Edit(*.py:*, *.ts:*)      # 文件類型限制
```

### 7.2 安全提醒系統

**文件位置：** `plugins/security-guidance/hooks/security_reminder_hook.py`

**檢測的 9 種危險模式：**

| 規則 | 檢測內容 | 風險類型 |
|------|---------|---------|
| `command_injection` | `child_process.exec()`, `os.system()` | 命令注入 |
| `eval_injection` | `eval()`, `new Function()` | 代碼執行 |
| `xss_attacks` | `dangerouslySetInnerHTML`, `.innerHTML =` | XSS |
| `pickle_deserialization` | Python `pickle` 模塊 | 反序列化攻擊 |
| `github_actions_workflow` | `.github/workflows/` | 工作流注入 |
| 其他 4 個 | ... | ... |

### 7.3 PreToolUse 驗證示例

```bash
# 路徑遍歷防護
if [[ "$file_path" == *".."* ]]; then
  echo '{"permissionDecision": "deny"}' >&2
  exit 2
fi

# 系統目錄保護
if [[ "$file_path" == /etc/* ]]; then
  echo '{"permissionDecision": "deny"}' >&2
  exit 2
fi

# 敏感文件提示
if [[ "$file_path" == *.env ]]; then
  echo '{"permissionDecision": "ask"}' >&2
  exit 2
fi
```

### 7.4 錯誤處理策略（Fail-Open）

```python
def main():
    try:
        result = engine.evaluate_rules(rules, input_data)
        print(json.dumps(result), file=sys.stdout)
    except Exception as e:
        # 錯誤時允許操作（fail-open）
        error_output = {"systemMessage": f"Error: {str(e)}"}
        print(json.dumps(error_output), file=sys.stdout)
    finally:
        sys.exit(0)  # 永遠不阻止操作
```

---

## 8. GitHub 自動化工作流

### 8.1 工作流清單（11 個）

| 工作流 | 觸發器 | 功能 | 關鍵文件 |
|--------|--------|------|---------|
| `claude.yml` | Issue/PR 評論 `@claude` | Claude Agent 自動執行 | `.claude/commands/*` |
| `claude-issue-triage.yml` | Issue 打開 | AI 自動分類和標籤 | 無腳本 |
| `claude-dedupe-issues.yml` | Issue 打開 | 檢測重複問題 | 無腳本 |
| `oncall-triage.yml` | 每 6 小時 | 標記關鍵問題 | 無腳本 |
| `auto-close-duplicates.yml` | 每日 9:00 UTC | 自動關閉重複 | `scripts/auto-close-duplicates.ts` |
| `stale-issue-manager.yml` | 每日 10:00 UTC | 管理陳舊問題 | 無腳本 |
| `lock-closed-issues.yml` | 每日 14:00 UTC | 鎖定已關閉問題 | 無腳本 |
| `log-issue-events.yml` | Issue 各種事件 | 事件日誌記錄 | 無腳本 |
| `remove-autoclose-label.yml` | PR/評論活動 | 移除自動關閉標籤 | 無腳本 |
| `backfill-duplicate-comments.yml` | 手動觸發 | 回填重複評論 | `scripts/backfill-duplicate-comments.ts` |
| `issue-opened-dispatch.yml` | 工作流分發 | 重新分發打開的問題 | 無腳本 |

### 8.2 問題生命週期管理

```
T+0d: 問題打開
  ├─ claude-issue-triage 自動分類
  └─ claude-dedupe-issues 重複檢測
         │
T+30d: 無活動警告
  ├─ stale-issue-manager 添加標籤
  └─ 發送警告評論
         │
T+60d: 自動關閉
  ├─ auto-close-duplicates 關閉重複
  └─ 其他問題標記為 stale
         │
T+67d: 自動鎖定
  └─ lock-closed-issues 防止垃圾評論
```

### 8.3 自動化腳本

**auto-close-duplicates.ts（270 行）：**
```typescript
// 核心邏輯
1. 獲取 3+ 天前打開的問題
2. 查找 Claude 生成的"重複檢測"評論
3. 驗證評論後無新活動
4. 檢查作者無👎反應
5. 自動關閉為重複狀態
6. 添加關閉評論
```

---

## 9. 閉源接口協議

### 9.1 插件載入接口

**Schema 驗證端點：**
```
.claude-plugin/marketplace.json
  ↓ schema 驗證
https://anthropic.com/claude-code/marketplace.schema.json  ← 閉源端
```

**Marketplace 配置格式：**
```json
{
  "$schema": "https://anthropic.com/claude-code/marketplace.schema.json",
  "name": "claude-code-plugins",
  "version": "1.0.0",
  "plugins": [
    {
      "name": "hookify",
      "version": "0.1.0",
      "source": "./plugins/hookify",
      "category": "productivity"
    }
  ]
}
```

**Plugin Manifest 格式：**
```json
{
  "name": "plugin-name",
  "version": "1.0.0",
  "description": "Plugin description",
  "author": {
    "name": "Author Name",
    "email": "author@example.com"
  }
}
```

### 9.2 自動發現機制

閉源核心會自動掃描以下目錄結構：

```
plugin-name/
├── .claude-plugin/plugin.json    # 元數據（必需）
├── commands/*.md                 # 斜線命令
├── agents/*.md                   # AI 代理
├── skills/*/SKILL.md             # 技能模塊
├── hooks/hooks.json              # Hook 配置
└── .mcp.json                     # MCP 服務器
```

### 9.3 多層通信架構

```
┌────────────────────────────────────────────────────────────────────┐
│                    Claude Code CLI（閉源）                          │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Layer 1: Plugin Discovery                                        │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ 掃描 marketplace.json → 加載 plugin.json → 註冊組件         │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  Layer 2: Hook System                                              │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ 事件觸發 → stdin JSON → Hook 腳本 → stdout JSON → 決策      │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  Layer 3: MCP Manager                                              │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ 加載 .mcp.json → 啟動服務器 → 工具發現 → 工具調用           │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  Layer 4: Frontmatter Parser                                       │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ 解析 *.md → 提取 YAML → 構建 Command/Agent/Skill 對象       │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  Layer 5: Environment Manager                                      │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ 注入 CLAUDE_* 變量 → 展開 ${} 引用 → 傳遞給子進程           │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 9.4 數據流圖

```
用戶輸入
    │
    ▼
┌─────────────────┐
│ UserPromptSubmit│ ← Hook 攔截點
│     Hook        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 命令/Agent 路由 │ ← Frontmatter 解析
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  PreToolUse     │ ← Hook 攔截點
│     Hook        │
└────────┬────────┘
         │
    ┌────┴────┐
    │ 決策    │
    ├─────────┤
    │ Allow   │──► 工具執行 ──► PostToolUse Hook
    │ Deny    │──► 阻止操作
    │ Ask     │──► 請求用戶確認
    └─────────┘
         │
         ▼
┌─────────────────┐
│     Stop        │ ← Hook 攔截點（可阻止退出）
│     Hook        │
└─────────────────┘
```

---

## 10. 配置管理系統

### 10.1 配置優先級

```
1. 專案級配置 (.claude/*.local.md)  ← 最高
2. 專案級命令 (.claude/commands/)
3. 插件配置 (plugin.json)
4. 全局用戶命令 (~/.claude/commands/)
5. 全局配置 (~/.claude/)
6. 插件默認值                        ← 最低
```

### 10.2 環境變量系統

**系統變量（由閉源核心注入）：**

| 變量 | 說明 | 示例值 |
|------|------|--------|
| `CLAUDE_PLUGIN_ROOT` | 插件根目錄 | `/path/to/plugin` |
| `CLAUDE_PROJECT_DIR` | 專案目錄 | `/path/to/project` |
| `CLAUDE_CONFIG_DIR` | 配置目錄 | `~/.claude` |
| `CLAUDE_ENV_FILE` | 環境文件 | `.env` |
| `CLAUDE_SESSION_ID` | 會話 ID | `abc-123` |

**其他控制變量：**

| 變量 | 說明 |
|------|------|
| `CLAUDE_CODE_SHELL` | Shell 選擇覆蓋 |
| `CLAUDE_BASH_NO_LOGIN` | 跳過登錄 Shell |
| `CLAUDE_CODE_AUTO_CONNECT_IDE` | IDE 自動連接 |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | 禁用非必要流量 |

### 10.3 命令定義格式

```markdown
---
description: Deploy to environment
argument-hint: [environment] [version]
allowed-tools: Bash(kubectl:*), Read
model: sonnet
---

Deploy to $1 environment with version $2

Configuration: @config/$1.json
Status: !`kubectl cluster-info`
```

**參數傳遞語法：**

| 語法 | 功能 |
|------|------|
| `$1`, `$2` | 位置參數 |
| `$ARGUMENTS` | 原始參數字符串 |
| `@file-path` | 文件內容插入 |
| `@$1` | 參數文件引用 |
| `` !`cmd` `` | 執行 Bash 命令 |
| `${CLAUDE_PLUGIN_ROOT}` | 插件路徑 |

---

## 11. 開發指南

### 11.1 創建新命令

```markdown
# .claude/commands/my-command.md
---
description: 我的命令
allowed-tools: Read, Bash(git:*)
model: sonnet
---

## 上下文
- 專案根目錄：!`pwd`

## 任務
執行我的任務...
```

### 11.2 創建新 Agent

```markdown
# plugins/my-plugin/agents/my-agent.md
---
name: my-agent
description: |
  Use this agent when...
  <example>
    context: ...
    user: "..."
    assistant: "..."
  </example>
model: sonnet
color: blue
tools: [Read, Grep, Bash]
---

你是一個代碼分析專家...
（系統提示）
```

### 11.3 創建新 Hook

**1. 編寫 Python 腳本：**

```python
# plugins/my-plugin/hooks/my_hook.py
#!/usr/bin/env python3

import json, sys

def main():
    input_data = json.load(sys.stdin)
    # 處理輸入
    result = {"systemMessage": "消息"}
    print(json.dumps(result))
    sys.exit(0)

if __name__ == '__main__':
    main()
```

**2. 配置 hooks.json：**

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash|Edit",
      "hooks": [{
        "type": "command",
        "command": "python3 ${CLAUDE_PLUGIN_ROOT}/hooks/my_hook.py",
        "timeout": 10
      }]
    }]
  }
}
```

### 11.4 創建新 Skill

```markdown
# plugins/my-plugin/skills/my-skill/SKILL.md
---
name: my-skill
description: 我的技能
version: 1.0.0
---

# 我的技能指南

完整的指導、最佳實踐和示例...
```

### 11.5 測試工具

| 工具 | 位置 | 功能 |
|------|------|------|
| `test-hook.sh` | `plugin-dev/skills/hook-development/scripts/` | Hook 單元測試 |
| `validate-hook-schema.sh` | 同上 | Hook 配置驗證 |
| `hook-linter.sh` | 同上 | 代碼品質檢查 |
| `validate-agent.sh` | `plugin-dev/skills/agent-development/scripts/` | Agent 結構驗證 |

**測試流程：**

```bash
# 創建測試輸入
./test-hook.sh --create-sample PreToolUse > test-input.json

# 執行測試
./test-hook.sh -v my-hook.sh test-input.json

# 驗證配置
./validate-hook-schema.sh ./hooks.json

# 代碼檢查
./hook-linter.sh ./my-hook.sh
```

---

## 12. 關鍵文件索引

### 12.1 核心實現文件

| 功能 | 文件路徑 | 行數 |
|------|---------|------|
| 規則評估引擎 | `plugins/hookify/core/rule_engine.py` | ~314 |
| 配置加載器 | `plugins/hookify/core/config_loader.py` | ~298 |
| PreToolUse Hook | `plugins/hookify/hooks/pretooluse.py` | ~75 |
| 安全提醒 | `plugins/security-guidance/hooks/security_reminder_hook.py` | ~281 |
| 重複關閉腳本 | `scripts/auto-close-duplicates.ts` | ~270 |

### 12.2 配置文件

| 文件 | 用途 |
|------|------|
| `.claude-plugin/marketplace.json` | 插件市場清單 |
| `plugins/*/.claude-plugin/plugin.json` | 插件元數據 |
| `plugins/*/hooks/hooks.json` | Hook 配置 |
| `.devcontainer/devcontainer.json` | 開發環境 |

### 12.3 命令和 Agent

| 類型 | 位置 |
|------|------|
| 專案命令 | `.claude/commands/*.md` |
| Feature Dev Agent | `plugins/feature-dev/agents/*.md` |
| Code Review | `plugins/code-review/commands/code-review.md` |
| PR Review Toolkit | `plugins/pr-review-toolkit/agents/*.md` |

### 12.4 技能文檔

| 技能 | 位置 |
|------|------|
| Hook 開發 | `plugins/plugin-dev/skills/hook-development/` |
| MCP 整合 | `plugins/plugin-dev/skills/mcp-integration/` |
| 命令開發 | `plugins/plugin-dev/skills/command-development/` |
| Agent 開發 | `plugins/plugin-dev/skills/agent-development/` |
| 插件結構 | `plugins/plugin-dev/skills/plugin-structure/` |

### 12.5 工作流

| 工作流 | 位置 |
|--------|------|
| Claude Agent | `.github/workflows/claude.yml` |
| 問題分類 | `.github/workflows/claude-issue-triage.yml` |
| 重複檢測 | `.github/workflows/claude-dedupe-issues.yml` |
| Oncall 分類 | `.github/workflows/oncall-triage.yml` |

---

## 13. 架構優缺點分析

### 13.1 優點

| 優點 | 說明 |
|------|------|
| **模組化設計** | 插件自包含，易於獨立開發和維護 |
| **聲明式配置** | Markdown + YAML，無需編碼即可定義行為 |
| **Hook 靈活性** | 9 個事件點，細粒度的行為控制 |
| **性能優化** | LRU 緩存、短路評估（70x 加速） |
| **零外部依賴** | Python 僅用標準庫，易於部署 |
| **自動發現** | 標準目錄結構自動加載組件 |
| **安全優先** | 多層驗證、fail-open 設計 |

### 13.2 缺點

| 缺點 | 說明 |
|------|------|
| **系統複雜性** | 多層抽象，初學者學習曲線陡峭 |
| **配置分散** | 配置文件分佈多處，不易集中管理 |
| **規則限制** | Hook 系統只支持 AND 邏輯（無 OR） |
| **性能開銷** | 每次工具執行都評估規則 |
| **文檔繁多** | 92+ Markdown 文件，容易造成混淆 |

### 13.3 最適合的用途

- 大型專案的自動化代碼審查
- 功能開發的結構化工作流
- GitHub Issue 的全自動管理
- 自定義 AI 行為控制（通過 Hook）
- 插件生態的擴展開發

---

## 附錄：快速參考

### A. 修復 Bug 的常見位置

| Bug 類型 | 查看位置 |
|---------|---------|
| Hook 問題 | `plugins/*/hooks/` |
| 命令問題 | `.claude/commands/` 或 `plugins/*/commands/` |
| Agent 問題 | `plugins/*/agents/` |
| 配置問題 | `plugin.json` 或 `hooks.json` |
| 腳本問題 | `scripts/` 或 `plugins/*/skills/*/scripts/` |

### B. 常用驗證命令

```bash
# 驗證 Hook 配置
./plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh hooks.json

# 測試 Hook
./plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh -v hook.sh input.json

# 驗證 Agent
./plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh agent.md

# 檢查代碼品質
./plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh hook.sh
```

### C. 調試技巧

```bash
# 啟用調試模式
claude --debug

# 查看日誌
tail -f ~/.claude/debug-logs/latest

# 檢查 Hook 輸出
echo '{"tool_name": "Bash", "tool_input": {"command": "ls"}}' | python3 hook.py
```

### D. 官方文檔連結

- https://docs.claude.com/en/docs/claude-code/overview
- https://docs.anthropic.com/en/docs/claude-code/hooks
- https://docs.anthropic.com/en/docs/claude-code/mcp

---

*此分析報告整合了之前所有分析文件的內容，涵蓋了 Claude Code 專案的架構、實現、安全、自動化和開發指南等所有主要方面。*
