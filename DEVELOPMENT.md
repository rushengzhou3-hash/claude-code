# 定制化开发指南

## 仓库结构

```
origin   → https://github.com/rushengzhou3-hash/claude-code  (我的 fork，日常推送到这里)
upstream → https://github.com/claude-code-best/claude-code   (上游原仓库，只读)

main        → 上游镜像分支，不做任何自己的改动
custom/main → 定制主分支，所有开发工作在这里进行
```

## 初始配置（已完成，仅供参考）

```bash
# 将原 origin（上游）重命名为 upstream
git remote rename origin upstream

# 添加自己的 fork 为 origin
git remote add origin https://github.com/rushengzhou3-hash/claude-code.git

# 创建定制主分支
git checkout -b custom/main

# 推送两个分支到自己的 fork
git push -u origin main
git push -u origin custom/main
```

## 日常开发

所有定制开发都在 `custom/main` 分支上进行：

```bash
git checkout custom/main

# 改代码、提交...
git add <文件>
git commit -m "feat: 你的改动描述"
git push origin custom/main
```

## 同步上游更新

不需要追每个 commit，按周或按里程碑批量评估即可。

### 第一步：拉取上游最新到 main

```bash
git checkout main
git fetch upstream
git merge upstream/main --ff-only
git push origin main  # 同步到自己的 fork
```

### 第二步：查看上游改了什么

```bash
# 看上游有哪些你还没有的 commit
git log custom/main..main --oneline

# 看某个具体文件上游改了什么
git diff custom/main..main -- src/query.ts
```

### 第三步：选择性 cherry-pick 需要的内容

```bash
git checkout custom/main

# 挑选单个 commit
git cherry-pick <commit-hash>

# 挑选一个范围
git cherry-pick abc123..def456
```

> 不需要的上游改动直接忽略，不要强行全量 merge。

## 文件分层策略

| 层级 | 文件示例 | 同步策略 |
|------|----------|----------|
| 完全自己的 | 新增 tools、自定义业务逻辑 | 不需要同步 |
| 重度改造 | `src/query.ts`、`src/screens/REPL.tsx` | 手动 diff 决定 |
| 轻度依赖 | `packages/`、`build.ts` | 偶尔同步 |
| 基本不动 | `src/types/`、stub 包 | 可直接 cherry-pick |

## 减少冲突的原则

- 尽量新增文件，而不是修改现有核心文件
- 新增 tool 放在独立目录 `src/tools/MyTool/`，不改 `src/tools.ts` 以外的地方
- 用环境变量或配置控制定制行为，避免硬编码
- 改动越靠近边缘越好，核心文件（`cli.tsx`、`main.tsx`）能不动就不动
