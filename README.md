# my-first-repo

在 Android 手机上跑 Minecraft Java 版的折腾记录，以及排查笔记。涉及 **MobileGlues** 渲染器、**Adreno / Mali** 驱动兼容性、**MC 26.3** 地形着色器编译失败这类问题；同时也是练习 git / GitHub 流程的地方。

> Keywords: Minecraft Java on Android · MobileGlues · Adreno 710 · Mali · GLSL / shader compile error · Vulkan 1.1 · FCL launcher · git basics

## 这个仓库存什么

- `git-basics.md` — git 最小可用流程笔记（暂存区 / 工作区 / 本地库 / 远程库）
- `notes/` — 真实问题排查记录
  - [MC 26.3 地形着色器编译失败排查笔记](notes/mobileglues-mc263-shader-issue.md)
  - [同配置下 FCL 能跑、ZPCL 必崩的对比分析](notes/fcl-vs-zpcl-mobileglues-comparison.md)
- `.gitignore` — 通用忽略规则（系统文件、密钥、日志、依赖产物）
- `LICENSE` — MIT，可自由使用与修改

## 结论（持续更新）

- 同一台设备（vivo V2410A / Adreno 710）、同一个 MC 26.3 原版、同一个 MobileGlues 渲染器：
  **FCL 1.3.4.8 能正常进世界，ZPCL 1.5.6 必崩。** 差异变量还在排查，见对比笔记。
- 本机的 Vulkan 相关路径全部不可用（硬件 Vulkan 1.1.0 + 内置 Turnip 不含 A710），
  MC 26.3 原生 Vulkan 后端、Zink、MobileGL 均无法启动。

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
- 本仓库已开启邮箱隐私，提交作者显示为 `359445943+lffggg@users.noreply.github.com` 匿名地址。
- 本地提交前确认身份设置正确（用 GitHub 给出的 noreply 地址，不要用真实邮箱）：

```bash
git config user.name  "lffggg"
git config user.email "359445943+lffggg@users.noreply.github.com"
```

## 许可证

[MIT](LICENSE) © 2026 lffggg
