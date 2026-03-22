# OpenClaw 版本降级指南

> 分析版本：2026.3.11 | 分析日期：2026-03-22

---

## 一、降级场景与风险提示

OpenClaw 处于快速迭代阶段（1.0 之前），版本更新可能伴随配置迁移或 breaking changes。
遇到以下情况时需考虑降级：

- 更新后 Gateway 无法启动
- 配置文件不兼容（deprecated keys / breaking changes）
- 已知问题影响生产使用
- 新版本引入非预期行为

> ⚠️ **降级前必读**：降级可能破坏配置文件。如果发生这种情况，参看 `openclaw doctor` 的迁移修复提示。

---

## 二、按安装方式的降级方法

### 2.1 全局安装（npm / pnpm）

#### 查看当前版本

```bash
npm view openclaw version
# 或
pnpm view openclaw version
```

#### 降级到指定版本

```bash
# npm
npm i -g openclaw@<version>

# pnpm
pnpm add -g openclaw@<version>
```

#### 降级后操作

```bash
# 必须运行 doctor 检查配置迁移
openclaw doctor

# 重启 Gateway
openclaw gateway restart

# 验证健康状态
openclaw health
```

---

### 2.2 源码安装（git checkout）

#### 方式 A：切换到指定版本标签

```bash
# 查看可用版本标签
git tag -l | grep -E '^v[0-9]'

# 切换到指定 stable 标签（如 v2026.2.1）
git checkout v2026.2.1

# 重装依赖 + 重建
pnpm install
pnpm build
pnpm ui:build

# 重启
openclaw gateway restart
openclaw doctor
```

#### 方式 B：按日期回滚到指定 commit

适用场景：没有对应版本标签，但知道"哪个日期的版本是好的"。

```bash
# 获取 origin/main 在指定日期的状态
git fetch origin
git checkout "$(git rev-list -n 1 --before="2026-01-01" origin/main)"

# 重装依赖 + 重建
pnpm install
pnpm build

# 重启 Gateway
openclaw gateway restart
```

#### 恢复到最新版本

```bash
git checkout main
git pull
pnpm install
pnpm build
openclaw gateway restart
```

---

### 2.3 通过 openclaw update 命令（仅限源码安装）

```bash
# 回退到 stable 频道（通常比当前稳定）
openclaw update --channel stable

# 指定版本（一次性，不修改持久配置）
openclaw update --tag v2026.2.1

# 预览降级操作（不实际执行）
openclaw update --dry-run
```

> 注意：`openclaw update` 在 npm 全局安装下会尝试通过包管理器更新；仅在检测到 git checkout 时才走源码回退流程。

---

## 三、更新频道切换

OpenClaw 三个更新频道：

| 频道 | 来源 | 说明 |
|------|------|------|
| `stable` | npm dist-tag `latest` | 最稳定，适合生产 |
| `beta` | npm dist-tag `beta` | 测试中，可能有 bug |
| `dev` | git `main` branch | 最新功能，不保证稳定 |

### 切换命令（对 npm 和 git 安装均有效）

```bash
# 切换到 stable
openclaw update --channel stable

# 切换到 beta
openclaw update --channel beta

# 切换到 dev
openclaw update --channel dev
```

当 `--channel dev` 时，OpenClaw 会确保存在 git checkout 并从 main 分支更新。

---

## 四、自动更新器配置与禁用

Auto-updater 默认关闭。如已开启且想禁用自动更新：

```json
{
  "update": {
    "channel": "stable",
    "auto": {
      "enabled": false
    }
  }
}
```

或者完全关闭启动检查：

```json
{
  "update": {
    "checkOnStart": false
  }
}
```

---

## 五、降级后的检查清单

| 步骤 | 命令 | 说明 |
|------|------|------|
| 1 | `openclaw doctor` | 检查配置迁移、修复建议 |
| 2 | `openclaw gateway restart` | 重启 Gateway |
| 3 | `openclaw health` | 验证 Gateway 健康状态 |
| 4 | `openclaw status` | 确认 session / channel 正常 |
| 5 | `openclaw logs --follow` | 观察运行时日志是否有错误 |

---

## 六、版本信息来源

- **当前安装版本**：2026.3.11
- **包管理器查看**：`npm view openclaw version` / `pnpm view openclaw version`
- **源码标签**：`git tag -l` 查看所有版本标签
- **更新状态**：`openclaw update status`
- **更新日志**：`openclaw/docs/CHANGELOG.md` 或 https://docs.openclaw.ai

---

## 七、遇到问题怎么办

1. 运行 `openclaw doctor` 并仔细阅读输出（通常会给出修复提示）
2. 查看 Gateway 日志：`openclaw logs`
3. 参考官方文档：https://docs.openclaw.ai
4. 社区支持：https://discord.gg/clawd

---

*文档由 Ayuya（暴风女王/16号技师）整理 | OpenClaw v2026.3.11*
