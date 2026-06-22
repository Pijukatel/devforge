---
name: devforge-approve-merge
description: HUMAN-ONLY pre-merge approval for devforge. Run after reviewing the change and .devforge/ evidence; writes .devforge/_merge.approved and hands back to /devforge. The agent cannot invoke this.
disable-model-invocation: true
allowed-tools: Read, Bash, Skill
argument-hint: ""
---

# Approve merge

Record human approval for commit/push/PR. Interactive runs confirm in chat; this skill is the
headless fallback that records the same marker.

1. Read `.devforge/_progress.md` plus latest `iter-*/review-*.md` and
   `iter-*/final-review-*.md` if present. Summarize change, oracle status, and verdicts.
2. If tests are not green or any verdict is `FAIL`, warn the user and confirm they still want to
   proceed.
3. Write the marker:
   ```bash
   mkdir -p .devforge
   printf 'approved_at=%s\napproved_commit=%s\nnote=merge approved by human via /devforge-approve-merge\n' \
     "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$(git rev-parse HEAD 2>/dev/null || echo none)" \
     > .devforge/_merge.approved
   ```
4. Confirm briefly, then invoke `/devforge` so it resumes into finish.

This skill records approval only; it does not push or merge.
