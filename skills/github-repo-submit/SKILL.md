# GitHub Repo Submit Skill

## 触发条件

当用户说"提交"、"push"、"同步到 GitHub"、"推到仓库"时激活。

## 前置要求

- GitHub Personal Access Token（存储在 memory 中或由用户直接提供）
- 目标 repo 地址

## 提交流程

### 1. 初始化 git（如果需要）

```bash
cd /root/.openclaw/workspace
git config user.email "<email>"
git config user.name "<name>"
```

### 2. 添加远程仓库

```bash
git remote add origin https://<TOKEN>@github.com/<owner>/<repo>.git
# 或如果已存在
git remote set-url origin https://<TOKEN>@github.com/<owner>/<repo>.git
```

### 3. 处理远程已有内容的情况

```bash
# 先 fetch
git fetch origin

# 如果有分歧，用 ours 策略合并（保留远程内容 + 我们的新文件）
git pull origin master --allow-unrelated-histories --no-rebase -X ours --no-edit
```

### 4. 添加文件（只加指定目录/文件）

```bash
git add <指定路径>   # 如 Analysis/
git status          # 确认只有要提交的文件
```

### 5. 提交并推送

```bash
git commit -m "<commit message>"
git push -u origin master
```

## 默认存放目录

`/Analysis/` —— 所有分析文档默认放这里，除非用户另有指定。

## 注意事项

- **不要** add 整个 workspace（包含 .openclaw/、.venv/、skills/ 等）
- 只提交有意义的文件（文档、分析报告等）
- Token 不要暴露在 commit message 中
