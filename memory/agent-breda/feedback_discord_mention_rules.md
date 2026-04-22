---
name: Discord mention rules for Breda
description: When to respond vs. stay silent in the Discord group channel
type: feedback
---

Only respond in the group channel when explicitly @mentioned with `@Breda` (Discord mention format). If a message says "Breda" without the @ symbol, treat it as context only and do not reply.

**Why:** The group channel delivers all messages regardless of mention. Responding without explicit mention creates noise and breaks the team's communication flow. Roy confirmed this via CLAUDE.md rules.

**How to apply:** Before composing any Discord reply, verify the inbound `<channel>` message contains `<@1488320292643143801>` (Breda's user ID) or an explicit @Breda mention. Exception: when Hawkeye ends a message with "Breda 구현 진행해도 돼" type statements, wait for Roy to formally delegate with @Breda.
