# codex-env 设计文档（Final）

**版本**：v1.0 — 2025-11-10  
**目标受众**：开发者/维护者（Rust）、项目维护者、CI 维护者

---

## 0. 背景与目标
Codex 的 MCP 配置默认是**全局一份**，在多项目并行开发时难以统一管理与切换。**codex-env** 旨在以 **Group** 为单位对 MCP（原称 Service/Server）进行分组管理，并将“项目级覆写 + 分组选择”后的 **有效配置（Effective Config）** 安全导出到目标端（Codex CLI / VS Code / 通用 JSON）。

> 术语调整：本文档将 **Service/Server** 统一更名为 **mcp**，命令行全部使用 `codex-env mcp ...`。

---

## 1. 名词与合并规则
- **Group**：一组 mcp 的集合，如 `python-dev`、`security-re`。
- **mcp**：一个 MCP 进程/端点定义（`id/command/args/env/transport/tags/enabled`）。
- **Effective Config**：全局仓库 + 项目覆写 + 选中分组叠加后的最终导出配置。

**合并优先级**（高→低）：
1) **project overrides**（项目配置）
2) **group defaults**（组内定义）
3) **global store**（全局仓库）

冲突策略：同名 `mcp.id` 以高优先级来源为准，`env` 采用**字典深度合并**（后写前），`args/tags` 采用**替换**而非拼接（避免顺序不确定）。

---

## 2. 配置路径与文件结构
### 2.1 全局配置（统一到 `~/.codex/venv/`）
```
~/.codex/venv/
├─ store.yaml            # 全局 groups & mcps 仓库
├─ backups/              # 导出/替换前自动备份
├─ adapters/             # 适配器模板与映射（可选）
└─ schema/               # JSON/YAML Schema（可选）
```

### 2.2 项目配置（建议统一至仓库内）
```
<repo>/.codex/venv/
└─ project.yaml          # 本项目启用的分组/覆写
```
> 说明：用户要求“工具配置文件放到 `~/.codex/venv/` 中”，全局仓库已迁移到此路径；项目侧为便于迁移与认知统一，建议同样放在 `<repo>/.codex/venv/`，不与其他工具冲突。

### 2.3 store.yaml（示例）
```yaml
version: 1
groups:
  python-dev:
    description: "MCP set for Python development"
    mcps: ["py-lsp", "yara-x"]
  security-re:
    description: "Reverse-engineering helpers"
    mcps: ["ida-mcp", "sigscan"]

mcps:
  py-lsp:
    id: "py-lsp"
    command: "uvx"
    args: ["python-lsp-server"]
    env: { "PYLSP_LOG": "1" }
    transport: "stdio"        # stdio|tcp|ws
    tags: ["lang", "python"]
    enabled: true

  yara-x:
    id: "yara-x"
    command: "yarax-mcp"
    args: []
    env: {}
    transport: "stdio"
    tags: ["security", "matching"]
    enabled: true
```

### 2.4 project.yaml（示例）
```yaml
use_groups: ["python-dev", "security-re"]
overrides:
  mcps:
    py-lsp:
      env: { "PYLSP_EXTRA": "on" }
```

---

## 3. CLI/TUI 交互设计

### 3.1 命令摘要
```
codex-env init [--group <name> ...] [--force]

codex-env group ls
codex-env group add <name>
codex-env group rm <name>
codex-env group rename <old> <new>
codex-env group show <name>

codex-env mcp ls [--group <name>]
codex-env mcp add --id <id> --cmd <exe> [--arg <a> ...] [--env K=V ...] \
                  [--transport stdio|tcp|ws] [--tag <t> ...] [--group <g>] [--enable|--disable]
codex-env mcp edit <id> [--cmd ...] [--arg ...] [--env ...] [--transport ...] [--enable|--disable]
codex-env mcp rm <id>
codex-env mcp toggle <id> [--on|--off]

codex-env use <group> [--project|--global] [--append|--exclusive]
codex-env status [--json]
codex-env validate [--strict]
codex-env diff [--target <raw|codex-cli|vscode>]

codex-env export [--target <raw|codex-cli|vscode>] [-o <path>|--stdout] [--dry-run] [--symlink <path>] [--backup]

codex-env activate --shell <bash|zsh|fish|powershell>

codex-env tui
codex-env doctor

# Windows: WSL 配置同步
codex-env wsl ls
codex-env wsl sync [--dist <name> ...] [--mode mirror|symlink] [--force]
codex-env wsl unsync [--dist <name> ...]
```

### 3.2 常见使用场景
```bash
# 初始化项目并绑定分组
codex-env init --group python-dev
codex-env use security-re --project --append

# 新增/编辑 mcp 并加入分组
codex-env mcp add --id yara-x --cmd yarax-mcp --transport stdio --tag security --group security-re
codex-env mcp edit py-lsp --env PYLSP_EXTRA=on

# 导出为 Codex CLI 可读取的 JSON（先看）
codex-env export --target codex-cli --stdout
# 真正写出并备份
codex-env export --target codex-cli -o ~/.codex/venv/mcp.json --backup

# Windows 下同步到 WSL 全部发行版（镜像复制）
codex-env wsl sync --mode mirror
```

### 3.3 TUI 草图与热键
```
┌ Groups (g) ────────────────┬ MCPs in Group (s) ─────────────────────┬ Details (d) ─────────┐
│ [*] python-dev              │ [x] py-lsp (stdio, enabled)            │ id: py-lsp           │
│ [ ] security-re             │ [x] yara-x (stdio, enabled)            │ cmd: uvx             │
│ [ ] ui-extras               │ [ ] ida-mcp (stdio, disabled)          │ args: [python-lsp...]│
└─────────────────────────────┴─────────────────────────────────────────┴──────────────────────┘
[f]ilter  [space] toggle  [a] add mcp  [e] edit  [g/G] add/rm group
[tab] switch pane  [x] export  [v] validate  [D] diff  [?] help  [q] quit
```

---

## 4. 导出适配器（Adapters）

### 4.1 目标与接口
- 目标：将 `Effective` 渲染为目标端格式，并以**原子写入**落盘，可选 `--stdout/--dry-run/--backup/--symlink`。
- 接口：
```rust
trait Adapter {
    fn name(&self) -> &'static str;           // "raw" | "codex-cli" | "vscode"
    fn render(&self, eff: &Effective) -> serde_json::Value;
    fn default_path(&self) -> Option<PathBuf>; // 可选：提供建议路径
}
```

### 4.2 支持的目标
- **raw**：通用 JSON，结构为：
```json
{
  "mcpServers": {
    "<id>": {"command":"...","args":["..."],"env":{"K":"V"},"transport":"stdio","enabled":true}
  }
}
```
- **codex-cli**：与 raw 等价或做轻微差异（根据 CLI 约定扩展字段，如 `stdio` 标志），默认输出到 `~/.codex/venv/mcp.json`。
- **vscode**：导出为 VS Code 插件需要的片段（如 settings JSON 片段或扩展自定义文件），默认仅 `--stdout`，由用户自行粘贴或 `-o` 写入。

### 4.3 diff/dry-run/backup/atomic
- `--dry-run`：不写文件，仅打印渲染结果或差异。
- `diff`：读取目标文件后与渲染结果做结构化 Diff（使用 `similar` 或 `json_patch` 比较）。
- **atomic**：写入 `tmpfile` → `fsync` → `rename`；
- **backup**：覆盖前复制到 `~/.codex/venv/backups/<timestamp>.json`。

---

## 5. Windows 上的 WSL 同步设计

### 5.1 目标
当 Codex/编辑器运行在 **WSL** 内部时，仍能读取到与 Windows 主机一致的 `~/.codex/venv/` 配置。提供两种模式：
- **mirror**：将 Windows `C:\Users\\<User>\\.codex\\venv` 内容**复制**到各发行版 `~/.codex/venv/`；
- **symlink**：在 WSL 侧创建**符号链接**指向 `/mnt/c/Users/<User>/.codex/venv`（性能更优，但环境权限/策略可能限制）。

### 5.2 架构与流程
- 枚举发行版：`wsl.exe -l -q` → 列出可用 Distro 名称。
- 针对每个 Distro 执行命令：
  - mirror：`wsl -d <D> -- bash -lc 'mkdir -p ~/.codex/venv && rsync -a --delete /mnt/c/Users/<User>/.codex/venv/ ~/.codex/venv/'`
  - symlink：`wsl -d <D> -- bash -lc 'rm -rf ~/.codex/venv && ln -s /mnt/c/Users/<User>/.codex/venv ~/.codex/venv'`
- `unsync`：
  - mirror：删除或保留由标记文件识别的镜像副本（可选 `--force`）。
  - symlink：`rm ~/.codex/venv` 并重建为目录（空）。

### 5.3 CLI 行为
```
codex-env wsl ls
codex-env wsl sync --mode mirror            # 同步到全部 Distro
codex-env wsl sync --dist Ubuntu-22.04 --mode symlink
codex-env wsl unsync --dist Ubuntu-22.04
```

### 5.4 限制与回退
- Windows 上创建到 WSL 的符号链接可能受权限策略影响；`symlink` 失败时自动回退到 `mirror` 并告知。
- `mirror` 下需在每次 `export` 后触发增量复制（可由 `export` 成功后自动触发 `wsl sync`）。

---

## 6. 校验（validate）与诊断（doctor）
- **validate**：
  - 重复 `mcp.id`、非法 `transport` 值、`command` 不存在/不可执行（跨平台 `.exe/.cmd/.bat` 兼容）。
  - 项目配置引用了不存在的 group/mcp。
- **doctor**：
  - 检查 `~/.codex/venv/` 可写、备份目录存在、WSL 可用性、`wsl.exe` 在 PATH。

---

## 7. 数据模型（Rust）
```rust
// model.rs
#[derive(Serialize, Deserialize, Clone, Debug)]
pub struct Mcp {
    pub id: String,
    pub command: String,
    #[serde(default)]
    pub args: Vec<String>,
    #[serde(default)]
    pub env: std::collections::BTreeMap<String, String>,
    #[serde(default)]
    pub transport: Transport, // default: stdio
    #[serde(default)]
    pub tags: Vec<String>,
    #[serde(default = "default_enabled")]
    pub enabled: bool,
}

#[derive(Serialize, Deserialize, Clone, Debug, Default)]
pub enum Transport { #[default] Stdio, Tcp, Ws }

#[derive(Serialize, Deserialize, Clone, Debug)]
pub struct Group { pub name: String, pub description: Option<String>, pub mcps: Vec<String> }

#[derive(Serialize, Deserialize, Clone, Debug)]
pub struct Store { pub version: u32, pub groups: BTreeMap<String, Group>, pub mcps: BTreeMap<String, Mcp> }

#[derive(Serialize, Deserialize, Clone, Debug, Default)]
pub struct ProjectOverrides { pub mcps: BTreeMap<String, McpOverride> }

#[derive(Serialize, Deserialize, Clone, Debug, Default)]
pub struct Project { pub use_groups: Vec<String>, pub overrides: ProjectOverrides }

#[derive(Serialize, Deserialize, Clone, Debug, Default)]
pub struct Effective { pub mcp_servers: BTreeMap<String, McpResolved> }
```
> 说明：`McpOverride` 仅允许修改 `env/args/transport/enabled/tags/command`（可配置）。

---

## 8. 模块划分与实现方案
- `cli.rs`：子命令解析（clap 衍生）。
- `model.rs`：数据结构与（反）序列化（serde）。
- `store.rs`：加载/保存/合并逻辑、校验。
- `export.rs`：适配器、diff、atomic 写入、backup、symlink。
- `wsl.rs`：WSL 枚举/同步（mirror/symlink）与错误回退。
- `tui.rs`：ratatui 三栏 UI；键位绑定；列表/筛选；调用 export/validate。
- `doctor.rs`：环境自检。

**关键点**：
- **Atomic**：`tempfile` → `write` → `flush+fsync` → `rename`。
- **Backup**：统一复制到 `~/.codex/venv/backups/`，文件名带时间戳。
- **Path**：`directories` crate；Windows 采用 `dirs` + 环境变量解析 Home。
- **Diff**：优先结构化（JSON）Diff，文本 Diff 仅作为可读性补充。
- **Logging**：`tracing` + `RUST_LOG`；所有日志英文。

---

## 9. 测试策略
- **单元测试**：
  - 合并规则：`store + project -> effective` 字段级断言（不使用易碎快照）。
  - 校验：重复 id、非法 transport、缺失 command、组内引用悬挂。
  - 导出：`--dry-run` 不落盘；`--backup` 生成备份；`atomic` 覆盖后内容可读。
- **平台测试**：
  - Windows 路径解析与 `which` 等效实现；`.exe/.cmd/.bat` 兼容检查；
  - WSL：`wsl ls/sync/unsync` 调用封装使用 **spy**（记录调用与参数），避免真实副作用。
- **端到端（可选）**：`assert_cmd` 驱动 CLI，用 `assert_fs` 构造临时仓库模拟项目场景。

---

## 10. 非目标与边界
- 不包含对 MCP 进程生命周期的直接管理（启动/重启）。
- 不联网获取“云模板”；仅本地文件系统操作。
- 不强制写入 VS Code 真实 settings.json，默认输出片段，由用户选择落地位置。

---

## 11. 路线图（R1→R3）
- **R1**：CLI 最小闭环（init/group/mcp/use/status/validate/export/tui），raw/codex-cli 适配器，Windows WSL `mirror` 模式。
- **R2**：`vscode` 适配器、TUI 就地编辑 env/args、`diff` 增强（按字段标注）。
- **R3**：WSL `symlink` 稳定化（策略检测与自动回退）、Schema 版本迁移工具、插件化 Adapter 加载。

---

## 12. 参考命令与示例
```bash
# 查看状态（JSON）
codex-env status --json | jq .

# 验证配置、查看与 VS Code 目标的差异
codex-env validate
codex-env diff --target vscode

# 导出+自动备份
codex-env export --target codex-cli -o ~/.codex/venv/mcp.json --backup

# Windows：同步到全部 WSL（失败自动回退到 mirror）
codex-env wsl sync --mode symlink || codex-env wsl sync --mode mirror
```

---

## 13. 架构示意（Mermaid）
```mermaid
flowchart LR
    CLI[CLI (clap)] --> CORE[Core Merge/Validate]
    TUI[ratatui] --> CORE
    CORE --> ADP[Adapters raw/codex-cli/vscode]
    CORE --> WSL[WSL Sync]
    ADP --> IO[Atomic IO + Backup]
    STORE[(~/.codex/venv/store.yaml)] --> CORE
    PROJECT[(<repo>/.codex/venv/project.yaml)] --> CORE
    IO --> TARGET[(mcp.json / VSCode snippet)]
    WSL --> TARGET_WSL[(~/.codex/venv on WSL)]
```

---
