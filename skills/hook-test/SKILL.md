---
name: hook-test
description: Test skill to verify frontmatter hooks work. Use when asked to test skill hooks.
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "bash -c \"echo 'SKILL HOOK FIRED' >> /tmp/skill-hook-test.log\""
---

# Hook Test Skill

This skill tests if skill-level hooks fire.

When invoked, run a simple bash command like `echo hello` to trigger the PreToolUse hook.

Then check if `/tmp/skill-hook-test.log` exists and contains "SKILL HOOK FIRED".

Report: did the skill hook fire?
