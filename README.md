# my-first-repo

Minecraft Java 版折腾与学习笔记仓库，同时也是练习 git / GitHub 流程的地方。

## 这个仓库存什么

- `git-basics.md` — git 最小可用流程笔记（暂存区 / 工作区 / 本地库 / 远程库）
- `.gitignore` — 通用忽略规则（系统文件、密钥、日志、依赖产物）
- `LICENSE` — MIT，可自由使用与修改

## 怎么用

```bash
# 克隆到本地
git clone https://github.com/lffggg/my-first-repo.git
cd my-first-repo

# 提交前的固定流程
git status
git add <文件名>
git commit -m "说明这次改了什么"
git push
```

## 安全提醒

- **私有令牌（PAT）绝不写进代码、绝不提交到仓库、绝不粘贴给任何人**。
  万一泄漏，立即去 GitHub → Settings → Developer settings 撤销，别指望删提交能找回。
- 本仓库已开启邮箱隐私，提交作者显示为 `@users.noreply.github.com` 匿名地址。
- 本地提交前确认身份设置正确：

```bash
git config user.name  "lffggg"
git config user.email "lffggg@users.noreply.github.com"
```

## 许可证

[MIT](LICENSE) © 2026 lffggg
