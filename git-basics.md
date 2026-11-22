# git 最小可用流程

## 四个区域

| 区域 | 在哪 | 命令 |
|---|---|---|
| 工作区 | 你正在编辑的文件 | — |
| 暂存区 | 待提交清单 | `git add` |
| 本地库 | `.git` 里的提交历史 | `git commit` |
| 远程库 | GitHub 上的仓库 | `git push` |

## 最小流程

```bash
git status                # 看现在在哪一步
git add 文件名 / git add -A
git commit -m "改了什么"
git push
git log --oneline        # 看历史
git pull --rebase        # 远程比本地上游时先拉
```

## 三条要诀

1. **`status` 永远不会害你**。不确定就多跑一遍。
2. **一次提交只做一件事**，提交信息写“做了什么”而不是“改了半天”。
3. **推送前先 `pull --rebase`**，冲突本地没解掉就不要 push。

## 参考资料

- 本地误操作搞乱后可用 `git reflog` 找回（保留约 90 天）
- 写错提交信息但还没推送：`git commit --amend`
