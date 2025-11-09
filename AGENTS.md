# AGENTS.md（工作流：PLAN → PATCH → TEST → COMMIT → NEXT）

> 原则：人类把关**设计**与**测试**；编码细节交给代理。对话使用**中文**，代码命名/注释/日志使用**英文**。

## A. 项目

### 1. PROJECT_GOAL
- **一句话**：实现基于 Rust 的 TUI/CLI 工具 **codex-env**，以 **Group** 管理 **mcp**，在多项目场景下生成并安全导出有效配置。
- **验收标准**：完成 `init/use/group/mcp/status/validate/export/tui/wsl` 的最小闭环；能在 Windows ↔ WSL 环境之间同步配置；`export --dry-run/--backup` 可用；单测全部通过。

### 2. DESIGN_STRATEGY
- **分层**：`cli (clap)` → `core (model+merge+validate)` → `adapters (export)` → `io (atomic+backup)` → `wsl (sync)` → `tui (ratatui)`。
- **路径**：全局配置统一到 `~/.codex/venv/`；项目配置 `<repo>/.codex/venv/`。
- **适配**：`--target raw|codex-cli|vscode`；不硬编码外部路径，支持 `-o/--symlink`。
- **平台**：Win/macOS/Linux 通用；Windows 支持 `wsl ls/sync/unsync`（`mirror|symlink`，带回退）。

### 3. SPECIAL_RULES
- 工作流中文；**代码/注释/日志英文**；提交信息英文。输出到终端的用户文案可中文但保持简洁。

---

## B. 工作流

### 1. PLAN（只读分析）
- **目标与成功标准**：完成 v1 骨架（CLI 命令解析、数据模型、合并、`export raw/codex-cli`、`validate`、TUI 占位、WSL `mirror`）。`codex-env status` 与 `export --stdout` 产物正确；`validate` 能拦截错误；Windows 下 `wsl ls/sync` 可运行（使用 spy）。
- **变更最小文件清单**：
  - `Cargo.toml`：`clap`, `serde_yaml`, `serde_json`, `anyhow`, `thiserror`, `directories`, `tempfile`, `ratatui`, `crossterm`, `similar`, `assert_fs`, `assert_cmd`, `predicates`。
  - `src/main.rs`, `src/cli.rs`, `src/model.rs`, `src/store.rs`, `src/export.rs`, `src/wsl.rs`, `src/tui.rs`, `src/doctor.rs`。
  - `tests/model_merge.rs`, `tests/export_raw.rs`, `tests/validate.rs`, `tests/wsl_spy.rs`。
- **测试计划**：
  - **正常**：
    1) `store.yaml + project.yaml -> effective` 字段断言；
    2) `mcp toggle/edit` 后 `export --stdout` JSON 符合预期；
    3) `export --dry-run` 不落盘；`--backup` 生成备份文件；
    4) `wsl ls` 返回 distro 列表（通过 spy）。
  - **错误**：
    1) 重复 `mcp.id`、非法 `transport`、缺失 `command` → `validate` 非零退出；
    2) `--symlink` 失败回退 `mirror`，并输出明确提示；
    3) `diff` 目标文件不可读/JSON 无效 → 友好错误信息；
  - **边界**：
    1) 空分组/空项目；
    2) Windows 路径与可执行文件后缀；
    3) 大量 `env` 键值合并稳定性；
  - **替身（fake/spy）**：
    - **FakeFS**：`assert_fs` 构造临时目录模拟 `~/.codex/venv/` 与项目目录；
    - **SpyWsl**：封装 `wsl.exe` 调用为 trait，测试注入记录参数/返回受控结果；
    - **SpyWhich**：命令存在性检查通过 trait 注入可控实现。

### 2. PATCH（最小变更）
- 实现数据模型与合并：`store + project -> effective`；
- 打通 CLI 子命令（空路由→最小实现）：`init/group/mcp/use/status/validate/export/tui/wsl`；
- `export` 支持 `--target raw|codex-cli`、`--stdout/-o/--dry-run/--backup/--symlink`，采用**原子写入**；
- `wsl` 子命令实现 `ls/sync/unsync`，`symlink` 失败自动回退 `mirror`（并打印原因）。

### 3. TEST（提交前的硬门禁）
- **小而快、确定性**：
  - `model_merge.rs`：构造 store/project，断言 `Effective`；
  - `validate.rs`：构造多种非法场景，断言错误码与关键信息；
  - `export_raw.rs`：解析导出 JSON，断言字段；`--dry-run` 不落盘；
  - `wsl_spy.rs`：注入 `SpyWsl`，验证 `ls/sync/unsync` 调用参数与回退逻辑；
- **顺序**：先跑子集，再全量；随机性固定种子；避免快照测试，优先结构化断言。

### 4. COMMIT（仅在全部通过后）
- **原子提交信息（英文）**：
```
feat(core): scaffold CLI, models, merge, export, and WSL sync (mirror)

- Add commands: init/group/mcp/use/status/validate/export/tui/wsl
- Implement merge & validation; adapters: raw, codex-cli; atomic write + backup
- Provide WSL ls/sync/unsync with symlink fallback to mirror
- Introduce TUI skeleton (three-pane)

Tests:
- model_merge, export_raw, validate, wsl_spy

Rollback:
- Restore previous config from ~/.codex/venv/backups/<ts>.json
```

### 5. NEXT（持续推进）
- **下一轮最小目标**：
  1) `vscode` 适配器完整化，`diff --target vscode`；
  2) TUI 内联编辑 env/args 并可即时 `export`；
  3) Windows 下 `symlink` 策略优化（检测开发者模式/管理员，自动选择）。
- **对应测试计划**：
  - 适配器输出结构校验；
  - TUI 修改后 `status/export` 一致；
  - `symlink` 路径/权限检测用 spy 复刻多种策略分支。

---

**附：示例 YAML/JSON**

`store.yaml` 与 `project.yaml` 见上。导出（raw/codex-cli）示例：
```json
{
  "mcpServers": {
    "py-lsp": {
      "command": "uvx",
      "args": ["python-lsp-server"],
      "env": {"PYLSP_LOG": "1", "PYLSP_EXTRA": "on"},
      "transport": "stdio",
      "enabled": true,
      "tags": ["lang","python"]
    },
    "yara-x": {
      "command": "yarax-mcp",
      "args": [],
      "env": {},
      "transport": "stdio",
      "enabled": true,
      "tags": ["security","matching"]
    }
  }
}
```

