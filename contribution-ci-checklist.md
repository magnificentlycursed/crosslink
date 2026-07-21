---
title: "Upstream contribution CI checklist"
tags: ["process", "ci"]
sources: []
contributors: ["unknown"]
created: 2026-07-21
updated: 2026-07-21
---

Every commit destined for dollspace-gay/crosslink must pass the exact CI lint job locally first: 'cargo clippy -- -D warnings -W clippy::unwrap_used -W clippy::expect_used' and 'cargo fmt --check' (from .github/workflows/ci.yml). The runner tracks current stable clippy, so new lints appear over time and develop itself can rot against them - fix findings for real, never with allow attributes. Local full-suite runs additionally need GIT_TERMINAL_PROMPT=0 GIT_ASKPASS=/usr/bin/false (gh#34 network tests) and expect 2 pre-existing bootstrap failures on hosts without a global git identity (gh#34 comment). Adopted 2026-07-21 after PR#32/PR#35 failed the linter pipeline on four pre-existing develop lint spots.
