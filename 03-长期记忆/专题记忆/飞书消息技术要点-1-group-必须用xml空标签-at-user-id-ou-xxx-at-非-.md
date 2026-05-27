---
name: 飞书消息技术要点：(1) Group @必须用XML空标签`<at user_id="ou_xxx"></at>`（非`<at>名</at>`），发text类型
description: 飞书消息技术要点：(1) Group @必须用XML空标签`<at user_id="ou_xxx"></at>`（非`<at>名</at>`），发text类型
type: note
created: 2026-05-27T18:54:10Z
updated: 2026-05-27T18:54:10Z
---

飞书消息技术要点：(1) Group @必须用XML空标签`<at user_id="ou_xxx"></at>`（非`<at>名</at>`），发text类型即可。 (2) 两个Chat ID：群聊oc_2f27b066780cad253745d1ea9af35697 / Home私聊oc_1b187944f8b6c76713697e6d7ec7cba9。 (3) Bot配置规则必须放config.yaml system_prompt第一行——AGENTS.md会被压缩遗忘。 (4) 自然回复会串话题——改用send_message发新消息到群主区。 (5) 禁止双发：选send_message就别自然回复。