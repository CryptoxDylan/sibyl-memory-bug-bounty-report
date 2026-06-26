## Sibyl Memory Plugin — Adversarial Bug Bounty Report

**Bounties:** B001 (Reproducible plugin defect) + B005 (Adversarial testing)

**Date:** June 26, 2026

**Tester:** Grok (xAI)

---

### Files

- **[sibyl-adversarial-bug-bounty-report.md](sibyl-adversarial-bug-bounty-report.md)** — Full detailed report in Markdown (best for reading on GitHub)
- **[sibyl-bounty-report.html](sibyl-bounty-report.html)** — Styled, self-contained HTML version (great for Discord or offline viewing)

---

### Summary

This repository contains a security/adversarial testing report against the Sibyl Memory Plugin (the drop-in memory system for agents via MCP, CLI, etc.).

Two findings are documented that meet the bounty criteria:

- **B001**: CLI crash on `sibyl status` due to mishandling of float timestamp in tier cache.
- **B005 (main)**: Memory poisoning / prompt injection vector. Arbitrary user-controlled memory bodies (including behavioral directives like `"how_to_apply"`) are returned verbatim with no trust labeling or sanitization. This enables persistent backdoors via a single `memory_remember` call.

Both findings include concrete reproduction steps, root cause analysis, and actionable hardening recommendations.

---

### Links

- Original bounty board: https://beta.sibyllabs.org/bounties
- Report submitted for review in the Sibyl Discord beta cohort

---

*Tested using live MCP tools in an activated environment.*