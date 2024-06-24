---
layout:     post
title:      "LangChain"
subtitle:   "LangChain使用"
date:       2024-06-24
author:     "ZBX"
header-img: "img/tag-bg.jpg"
tags:
    - LangChain
---

## Agent 

### Agent提示词模板样例

```
请尽你所能回答一下问题，但要像xx那样说话。你可以使用一下工具:
search: 当你需要回答当前事件问题时很有用。

请使用以下格式：
问题：你必须回答的输入问题
思考：你应该总是思考要做什么
行动：要采取的行动，应该是[search]之一
行动输入：行动的输入
观察：行动的结果......（这个思考/行动/行动输入/观察可以重复N次）
思考：我现在知道最后的答案了
最终答案：对于原始输入问题的最终答案

Question： {input}
{agent_scratchpad}
```

Agent组件遵循的是ReAct框架的策略（思考/行动/行动输入/观察可以重复N次）

## 基于本地知识库问答的实现原理

![image-20240624230618815](https://s2.loli.net/2024/06/24/jQPhpCtOlIRcgHw.png)