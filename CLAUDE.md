# CLAUDE.md

## Commands

```bash
# 安装 / 同步依赖
uv sync

# Lint 与类型检查
uv run ruff check src tests scripts
uv run mypy src

# 测试
uv run pytest tests/unit -v           # 单元（快，无 daemon）
uv run pytest tests/integration -v    # fixture 自行拉起 daemon
uv run pytest tests/ -v               # 全量

# 单个用例
uv run pytest tests/unit/test_xxx.py::test_yyy -v

# 手动启动 daemon / 探活
uv run pcd
uv run pc ping
```

## Architecture

仓库从空开始，架构随实现推进更新到本节。规划中的形态：daemon + 客户端的双进程本地 Agent 运行时；传输方式、线协议、上下文治理、权限模型、前端形态等关键选择全部记录在 `docs/decisions.md`（每项含 KamaClaude 的原选择、替代方案与当前状态）。

规则：**某项决策在 `docs/decisions.md` 里定下来之前，不要在代码里隐式引入它。**

## Code style

- 类型注解齐全，mypy strict 兜底
- 注释只写代码本身说不清的东西：约束、踩过的坑、为什么不用显然的方案
- 公共 API 一行 docstring；不逐函数强制注释
- 测试命名 `test_<被测行为>`，断言表达不变式，不复述操作步骤
- 提交小步快跑：一行说清做了什么，必要时补一行为什么；不攒巨型 commit

这节是本仓库自己的规范，想调整口味直接改这里。

## 工程约束

- Python 3.12（`>=3.12,<3.13`），uv + hatchling，src 布局：包在 `src/private_claude/`
- 入口点：`pc`（CLI）、`pcd`（daemon）——已定（见 `docs/decisions.md` D19；`pc-tui` 随 D16 的 rich CLI 决策取消）
- ruff（E/F/I/UP，行长 100）、pytest（`asyncio_mode = "auto"`；`integration` 标记的用例需要真实 API key）
