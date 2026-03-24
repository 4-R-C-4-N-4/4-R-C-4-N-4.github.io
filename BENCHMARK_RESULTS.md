# Agent Benchmark Results: Rust Task Manager — Fix & Test

**Benchmark:** Debug, fix, and test a broken Rust CLI task manager (8 known bugs, 12 required test behaviors)  
**Runs per agent:** 3  
**Acceptance criteria:** `cargo build` ✓, `cargo test` ≥10 passing / 0 failing, `cargo clippy` 0 warnings, all 8 bugs fixed

---

## Summary Table

| Metric             | HERMES Run 1 | HERMES Run 2 | HERMES Run 3 | SLATE Run 1  | SLATE Run 2  | SLATE Run 3  |
|--------------------|--------------|--------------|--------------|--------------|--------------|--------------|
| **Time**           | 1 min        | 1 min        | 2 min        | 15 min       | 8 min        | 5 min        |
| **Tokens**         | 23.8k        | 23.2k        | 27.8k        | 1,200k       | 1,000k       | 685k         |
| **Models Used**    | 1            | 1            | 1            | 3            | 3            | 3            |
| **Tests Passing**  | 28           | 25           | 26           | 20           | 37           | 27           |
| **Build**          | ✅ Pass       | ✅ Pass       | ✅ Pass       | ✅ Pass       | ✅ Pass       | ✅ Pass       |
| **Clippy Clean**   | ✅ 0 warnings | ✅ 0 warnings | ✅ 0 warnings | ✅ 0 warnings | ✅ 0 warnings | ✅ 0 warnings |
| **Cost**           | —            | —            | —            | $2.56        | $1.45        | $1.19        |
| **API Requests**   | —            | —            | —            | 106          | —            | —            |

---

## Averages

| Metric            | HERMES (avg)  | SLATE (avg)   | Delta                  |
|-------------------|---------------|---------------|------------------------|
| **Time**          | **1.3 min**   | 9.3 min       | HERMES ~7x faster      |
| **Tokens**        | **24.9k**     | 961.7k        | HERMES ~39x fewer      |
| **Tests Written** | **26.3**      | 28.0          | Comparable             |
| **Build + Clippy**| 3/3 pass      | 3/3 pass      | Tied                   |
| **Cost**          | —             | ~$1.73/run    | —                      |

---

## Key Takeaways

1. **Both agents passed all acceptance criteria** across all 3 runs — clean builds, zero clippy warnings, all tests passing, minimum test threshold (≥10) exceeded.

2. **HERMES is dramatically more efficient.** It completed the task in ~1 min using ~25k tokens on a single model, compared to SLATE's ~9 min average across 3 models consuming ~960k tokens per run.

3. **Token efficiency gap is ~39x.** HERMES used roughly 25k tokens per run vs. SLATE's ~962k — nearly two orders of magnitude difference.

4. **Time efficiency gap is ~7x.** HERMES averaged 1.3 minutes vs. SLATE's 9.3 minutes.

5. **Test coverage is comparable.** Both agents wrote thorough test suites exceeding the 12 required test behaviors. HERMES averaged 26.3 tests, SLATE averaged 28.0 — effectively equivalent.

6. **SLATE had higher run-to-run variance.** Time ranged from 5–15 min and tokens from 685k–1.2M. HERMES was more consistent (1–2 min, 23.2k–27.8k tokens).

7. **Cost asymmetry.** SLATE's multi-model approach averaged ~$1.73 per run. HERMES cost data was not recorded but given the ~39x token reduction, it would be a fraction of SLATE's cost.

---

## Methodology Notes

- Both agents received identical starter code and spec (SPEC.md)
- Task: fix 8 bugs (4 compile errors, 4 logic bugs) and write ≥12 unit tests
- All runs in independent fresh environments
- SLATE uses a 3-model architecture; HERMES uses a single model
