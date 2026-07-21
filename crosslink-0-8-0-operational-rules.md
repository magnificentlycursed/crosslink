---
title: "Crosslink 0.8.0 operational rules (upgrade hold and defect guards)"
tags: ["operations", "upstream"]
sources: []
contributors: ["unknown"]
created: 2026-07-21
updated: 2026-07-21
---

HARD CONSTRAINT (operator ruling 2026-07-20, decision on vsdd-cli #597): the host binary stays at crosslink 0.8.0 for vsdd-cli and mdatron. Never upgrade to 0.9.x, never run 'crosslink migrate hub-v3' on these hubs. Retest trigger: dollspace-gay/crosslink #4 #5 #7 #8 #11 #12 all closing (tracked locally as epic #3).

Defect guards while on 0.8.0:
- 'issue close' and 'archive add' can report success without persisting (gh#29, gh#30; local epic #10). Re-read state after these operations; never treat the success message as the record.
- After any 'crosslink sync' that promotes issue IDs, re-verify the session work binding and locks; promotion strands them (forecast-bio#653-era defect, live at 0.8.0, no dollspace-gay issue yet; local #14).
- Titles from '-q' output are truncated to 40 chars (gh#14; local #11). Use --json whenever titles or exact text matter.
- Never suppress crosslink WARN output; no '2>/dev/null' on crosslink commands. WARN dismissal was the proximate cause of the 2026-05-28 identity leak (see gh#27, gh#28 for the filed aftermath).
