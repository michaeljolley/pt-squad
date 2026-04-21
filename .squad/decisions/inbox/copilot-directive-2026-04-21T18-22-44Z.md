### 2026-04-21T18:22:44Z: Tag overflow fix approach for #38317
**By:** Michael Jolley (via Copilot)
**What:** Two-pronged fix for Command Palette list item tag overflow:
1. Prioritize title text — title always gets layout priority, tags get remaining space. Handle overflow with ellipsis appropriately.
2. Cap visible tags at 3 — if more than 3 tags, show only 3 and append a `[+ N]` badge indicating how many more exist.
**Why:** User design decision for issue #38317 — ensures title is always readable while keeping tags useful.
