# OpenViking 未时装

## 序言

> date：2026/11/8

由大肥鱼创作，我负责校对。

本篇回答，有哪些 OpenViking 能提供的功能，我暂时还没用上。

## 目前不具备的能力

以下要升级/额外启动才有：

**VikingBot**（在启动时加 --with-bot 就可启动 VikingBot）：
- OpenViking 自带的对话 Agent，跑在服务端进程里（--with-bot），内置了检索、读取、记忆这套工具，等于「不装任何 harness 也能直接跟这个库聊天」。

**规则 Rule（Beta）**：
- 把规范编译成可检索的规则条目，存在 memories/rules。执行任务前按当前任务内容检索「这次适用哪几条」，带 0.65 的概率阈值。

**Agent 经验 Experience**：
- memories/ 下的 experiences、trajectories、cases、tools 几类，记录「这类任务上次怎么做、哪步踩坑」，配套 search_experience 在执行类任务前先查。

**上下文编译（Compile）**：
- 把源资料编译成产物：LLM Wiki、知识图谱、日报、知识蒸馏、记忆整合。需要 Compile Runtime（执行环境 + 模型）才跑得起来。

**ov CLI**：
- 命令行客户端（npm 包），不是服务端能力。ov add-resource --wait、ov task status、ov find、ov read、ov tree、导入导出都靠它。

| 项目	|  缺了会少什么 | 现在需要吗 |
|---|---|---|
| VikingBot	| Studio 的 VikingBot 页是空的，ov chat 不可用。不影响到 DSH，因为两者读的是同一个库	| 不需要。已有 DSH，能力还更强。它唯一的价值是「验证这个库脱离 DSH 也能独立用」 |
| 规则 Rules（Beta） |	 少了「任务前自动查规则」这一步。现在的 preferences 仍然会被注入，只是不判定适用性	| 不急。个人使用 preference 够用；Rules 是给团队做硬约束的（比如「删文件前必须确认」）|
| Agent 经验 Experience	| 少一层「复盘复用」。事件、偏好这些普通记忆照常工作 |	暂时不需要。当需要反复做同类实验或部署时，它才会有明显价值 |
| 上下文编译 Compile	| 只能「查」不能「生成成篇的东西」。但注意：智能体可以自己读资料再写，不一定非得走 Compile	| 不需要。目前产出主要靠对话生成，Compile 是给「定期自动出报告」这类场景的 |
| ov CLI	| 只能用 Studio 和对话，没法脚本化、定时、批量	| 可选但值得装。成本约一分钟，ovcli.conf 已经配好，装完 ov health 直接通 |
