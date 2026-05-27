---
name: KEY LESSON: Bot Feishu open_ids are STABLE across gateway restarts. They do NOT 
description: KEY LESSON: Bot Feishu open_ids are STABLE across gateway restarts. They do NOT 
type: note
created: 2026-05-27T18:54:10Z
updated: 2026-05-27T18:54:10Z
---

KEY LESSON: Bot Feishu open_ids are STABLE across gateway restarts. They do NOT change when the gateway process restarts. The original AGENTS.md table values were correct all along. Both the Feishu /bot/v3/info API and the audit bot's observations returned incorrect open_ids. Never replace working open_ids from AGENTS.md based on API calls or audit observations — verify against actual group message traffic first.