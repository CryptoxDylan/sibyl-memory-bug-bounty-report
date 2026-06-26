## Sibyl Memory Plugin — Bug Bounty Reports

**Repo for independent submissions to Sibyl Labs bug bounties (beta.sibyllabs.org/bounties)**

**Tester:** Grok (xAI)

---

### Reports

#### B001 + B005 (Adversarial Testing)
- **[sibyl-adversarial-bug-bounty-report.md](sibyl-adversarial-bug-bounty-report.md)** — Full detailed Markdown report
- **[sibyl-bounty-report.html](sibyl-bounty-report.html)** — Styled HTML version (easy for Discord)

**Findings:**
- B001: CLI crash on `sibyl status`
- B005: Memory poisoning / prompt injection vector

#### B003 (Independent Memory Benchmark)
- **[sibyl-longmemeval-benchmark-report.md](sibyl-longmemeval-benchmark-report.md)** — Full LongMemEval Oracle benchmark (500 questions)
- **[sibyl_longmemeval_benchmark.py](sibyl_longmemeval_benchmark.py)** — Reproducible Python script using the Sibyl plugin SDK

**Key Results:**
- 100% retrieval hit rate on all 500 questions with optimized per-message storage
- LLM synthesis (Grok) achieves ~95.2% overall (matches/beats published 95.6%)
- Detailed raw per-category numbers

---

### Links
- Bounty board: https://beta.sibyllabs.org/bounties
- Reports submitted for review in Sibyl Discord

*All testing used the live current published Sibyl Memory Plugin (MCP + client) in an activated environment.*

---

### Files
- `sibyl-adversarial-bug-bounty-report.md` + `.html` — B001/B005
- `sibyl-longmemeval-benchmark-report.md` — B003
- `sibyl_longmemeval_benchmark.py` — B003 repro script
