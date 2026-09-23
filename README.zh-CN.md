<div align="right">

[English](README.md) | 简体中文

</div>

# dsh-plugin-upgrade

**一套支持无头运行的 SOP 技能：在 dsh 升级后，把 DeepSeek Harness 插件升级到新版本。**

dsh 在 0.x 阶段迭代极快，插件接缝经常被破坏。这个技能把"升级后插件失效"从考古式排查变成可重复的流程：先取证，再对照已安装源码逐条复查接缝，小步修复、每次修复单独提交，最后走验证阶梯、打版本同步 tag。

已在 dsh-plugin-job-panel 的升级 `0.1.5-rc.2 → 0.1.7-alpha.2`（2026-09-23）上实战验证：官方重构弹窗后 React key 从行 `<li>` 移到了组件 fiber 上，本流程几分钟内就从已安装源码的证据中定位了问题。

[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](./LICENSE)
![verified](https://img.shields.io/badge/verified-dsh%200.1.7--alpha.2-blue)

## 流程一览

| 步骤 | 内容 |
|---|---|
| §0 原则 | 取证先于改码；已装源码=唯一事实；修复兼容新旧形态；小步提交；绝不从 dsh web 内部重启它 |
| §0.5 无头模式 | 所有询问按钉死的推荐默认值自动决策；外向动作（publish / push / 重启）只准备不执行 |
| §1 范围 | 自动盘点已安装的第三方插件与 DSH 相关技能 |
| §2 源码与版本 | 定位已安装 dsh 与上游 tag，缺源码则 clone；附带任务：dev-notes 技能自身过时则顺手复查 |
| §4 逐插件升级 | 先读诊断文件（哪半边坏了？），找插件的验证基线，只 diff 依赖面的 dsh 包（`npm pack` 新旧对比），逐条判决接缝 |
| §5 报告 | 失效范围 / 根因（附新旧证据）/ 修复方式 / 验证方法 |
| §6 收尾 | 验证阶梯（测试 → bundle 冒烟 → 带 cookie 探活 → 点击验证）、逐修复提交、强烈建议打 `verified-dsh/vX.Y.Z` tag |

完整流程在 [SKILL.md](SKILL.md) —— 该文件即技能本体，本 README 只是门面。

## 无头模式

SOP 里每个询问点都有推荐答案，因此可以无人值守运行：说"无头 / headless / 不要问我"，所有决策按默认值自动做出并记录，最终报告可完整复盘。永不自动执行的红线：`npm publish`、`git push`、重启 `dsh web` —— 只准备精确命令交给你。决策表见 SKILL.md §0.5。

## 安装

前置要求：无（技能就是单个 markdown 文件）。

```sh
# 通过 skills CLI
npx -y skills add cholf5/dsh-plugin-upgrade
# 或手动
git clone https://github.com/cholf5/dsh-plugin-upgrade.git ~/.agents/skills/dsh-plugin-upgrade
```

与 [dsh-plugin-dev-notes](https://github.com/cholf5/dsh-plugin-dev-notes)（双面插件开发实战笔记，本 SOP 的"对照已装源码验证"方法论即出自它）配合使用效果最佳。

## 用法

在 agent 中加载技能后：

- 直接点名要升级的插件 —— 立即开始；
- 裸加载 —— 它盘点你的 profile 后询问升级哪些；
- 加上无头措辞 —— 按推荐默认值全自动跑完。

## 许可

[MIT](./LICENSE)
