---
name: devforge-approve-design
description: HUMAN-ONLY design-gate approval for devforge. Run after reviewing .devforge/2-design.md and .devforge/_panel.json; records the panel in _state.json, writes .devforge/_design.approved, and hands back to /devforge. The agent cannot invoke this.
disable-model-invocation: true
allowed-tools: Read, Bash, Skill
argument-hint: ""
---

# Approve design

Record human approval of the design and review panel so implementation may begin.

1. Read `.devforge/2-design.md` and `.devforge/_panel.json`; stop if either is missing.
2. Summarize the approach, planned changes, reviewers, final reviewers, limits, and panel
   reason in 3-5 lines.
3. Copy the approved panel into `_state.json`:
   ```bash
   python3 - <<'PY'
   import json
   from pathlib import Path
   state_path = Path(".devforge/_state.json")
   panel = json.loads(Path(".devforge/_panel.json").read_text())
   for key in ("reviewers", "final_reviewers", "inner_iterations", "final_review_rounds"):
       if key not in panel:
           raise SystemExit(f"_panel.json missing required key: {key}")
   state = json.loads(state_path.read_text()) if state_path.exists() else {}
   state["panel"] = panel
   state["phase"] = "inner-loop"
   state["iteration"] = 1
   state_path.write_text(json.dumps(state, indent=2, sort_keys=True) + "\n")
   PY
   ```
4. Write the marker:
   ```bash
   mkdir -p .devforge
   printf 'approved_at=%s\napproved_commit=%s\nnote=design approved by human via /devforge-approve-design\n' \
     "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$(git rev-parse HEAD 2>/dev/null || echo none)" \
     > .devforge/_design.approved
   ```
5. Confirm briefly, then invoke `/devforge` so it resumes into implementation.

Do not edit source here.
