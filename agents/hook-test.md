---
name: hook-test
description: Test agent to verify frontmatter hooks work. Use when asked to test hooks.
tools: Read, Bash
model: haiku
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "bash -c \"echo 'AGENT HOOK FIRED' >> /tmp/agent-hook-test.log\""
---

# Hook Test Agent

You are a test agent. When invoked, run a simple bash command like `echo hello` to trigger the PreToolUse hook.

After running the command, check if `/tmp/hook-test.log` exists and contains "HOOK FIRED".

Report back: did the hook fire?
