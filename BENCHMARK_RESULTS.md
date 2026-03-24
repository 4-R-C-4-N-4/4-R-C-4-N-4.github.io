# Agent Benchmark Results: Rust Task Manager — Fix & Test

**Task:** Debug, fix, and test a broken Rust CLI task manager  
**Scope:** 8 known bugs (4 compile errors, 4 logic bugs), comprehensive test suite (≥12 required behaviors)  
**Runs per agent:** 3 (independent fresh environments)  
**Acceptance criteria:** `cargo build` ✓, `cargo test` ≥10 passing / 0 failing, `cargo clippy` 0 warnings

### Model Configuration

| Agent    | Architecture         | Models                                          |
|----------|----------------------|-------------------------------------------------|
| **HERMES** | Single model, direct execution | Claude Opus                           |
| **SLATE**  | Multi-model subagent orchestration | Claude Opus (orchestrator) + Claude Sonnet (worker) + Claude Haiku (reads) |

---

## Results At A Glance

| Metric             | HERMES Run 1 | HERMES Run 2 | HERMES Run 3 | SLATE Run 1  | SLATE Run 2  | SLATE Run 3  |
|--------------------|--------------|--------------|--------------|--------------|--------------|--------------|
| **Wall Time**      | 1 min        | 1 min        | 2 min        | 15 min       | 8 min        | 5 min        |
| **Tokens**         | 23.8k        | 23.2k        | 27.8k        | 1.2M         | 695k         | 428k         |
| **Cost**           | —            | —            | —            | $2.56        | $1.45        | $1.14        |
| **Models Used**    | 1            | 1            | 1            | 3            | 3            | 3            |
| **Tests Passing**  | 28           | 25           | 26           | 20           | 37           | 27           |
| **Build**          | ✅ Pass       | ✅ Pass       | ✅ Pass       | ✅ Pass       | ✅ Pass       | ✅ Pass       |
| **Clippy Clean**   | ✅ 0 warnings | ✅ 0 warnings | ✅ 0 warnings | ✅ 0 warnings | ✅ 0 warnings | ✅ 0 warnings |
| **Bugs Fixed**     | All 8 + 1    | All 8 + 1    | All 8 + 1    | All 8 + 2    | All 8 + 1    | All 8 + 1    |

*"+1" = clippy's `new_without_default` lint (Default impl). "+2" = Default impl + sort order fix.*

---

## Averages

| Metric            | HERMES (avg)  | SLATE (avg)   | Delta                  |
|-------------------|---------------|---------------|------------------------|
| **Wall Time**     | **1.3 min**   | 9.3 min       | HERMES ~7x faster      |
| **Tokens**        | **24.9k**     | 774k          | HERMES ~31x fewer      |
| **Tests Written** | 26.3          | 28.0          | Comparable             |
| **Build + Clippy**| 3/3 pass      | 3/3 pass      | Tied                   |
| **Cost**          | —             | ~$1.72/run    | —                      |

---

## Architecture & Workflow Comparison

### HERMES — Single Model, Direct Execution

HERMES uses a single model with direct tool calls. Typical workflow:

```
1. Read all files (5 parallel reads)          ~4s
2. Fix all source files (5 parallel writes)   ~3s
3. cargo build → cargo test → cargo clippy    ~5s
4. Write output files (2 parallel writes)     ~1s
                                        Total: ~13s active
```

Key characteristics:
- **No subagents.** All work done in a single context with parallel tool batches.
- **One-pass fix.** Reads all files, identifies all bugs, writes all fixes at once.
- **Single verification.** Build/test/clippy run once, pass on first try.
- **Zero retries.** No redundant re-reads or re-verifications across all 3 runs.
- Run 3 had a minor code-exec error but recovered immediately.

### SLATE — Multi-Model Subagent Architecture

SLATE uses 3 models (claude-sonnet-4.6 as orchestrator/worker, claude-haiku-4.5 for reads) and delegates every operation to subagents.

Typical workflow (Run 2 — its cleanest run):
```
1. Spawn 2 subagents to read SPEC + explore dir      ~13s
2. Spawn 1 subagent to list directory tree            ~3s
3. Spawn 1 subagent to move files into src/           ~3s
4. Spawn 4 subagents to fix 4 source files in parallel ~12s
5. Spawn 1 subagent to write tests.rs                 ~37s
6. Spawn 1 subagent to build/test/clippy              ~16s
7. Spawn 1 subagent to write output files             ~6s
                                               Total: ~90s active
```

Key characteristics:
- **Subagent-per-task.** Even simple operations (reading a file, running `ls`) spawn subagents.
- **Multi-model routing.** Uses cheaper haiku for file reads, sonnet for edits — adds orchestration overhead.
- **Redundant verification.** Run 1 spent ~4 extra verification rounds ("the system keeps indicating the task isn't complete"), burning 198k tokens on a single re-check subagent.
- **Subagent failures.** Run 3 had 2 interrupted subagents trying to read `manager.rs`, requiring 3 attempts.
- **High variance.** Run 1 took 15 min / $2.56; Run 3 took 5 min / $1.14 — 3x spread.

---

## Behavioral Observations

### Bug Detection
Both agents identified all 8 spec bugs on first read. Both also found:
- The sort order bug in `get_tasks_by_priority()` (ascending → descending)
- Clippy's `new_without_default` lint → added `Default` impl for `TaskManager`

### Test Quality
Both wrote thorough test suites exceeding the 12 required behaviors:
- **HERMES** averaged 26.3 tests — covered priority scores/display, task lifecycle, manager operations, summary formatting, serialization, edge cases
- **SLATE** averaged 28.0 tests — similar coverage plus additional edge cases (100-task unique ID stress test in Run 2, double-complete in Run 1)

Test count varied between runs for both agents, which is expected given non-deterministic generation.

### Failure Modes
- **HERMES:** No failures across 3 runs. Run 3 had a code-exec error (Python script) but recovered immediately by falling back to direct tool calls.
- **SLATE:** No correctness failures, but significant efficiency failures:
  - Run 1: Verification spiral — 4 extra rounds of `cargo build/test/clippy` + file re-reads, 106 API requests total
  - Run 3: 2 interrupted subagents when trying to read `manager.rs` (subagent got confused and didn't surface file contents as evidence)

---

## Key Takeaways

1. **Both agents passed all acceptance criteria** across all 3 runs — clean builds, zero clippy warnings, all tests green, all 8 bugs fixed.

2. **HERMES is dramatically more efficient.** ~1.3 min / ~25k tokens vs. SLATE's ~9.3 min / ~774k tokens. The token gap is ~31x.

3. **Single-context direct execution beats multi-agent orchestration** for this task size. HERMES's approach of reading everything into one context, reasoning once, and writing all fixes in parallel avoided the overhead of subagent spawning, context serialization, and evidence surfacing.

4. **SLATE's subagent architecture adds fragility.** Interrupted subagents, evidence-surfacing failures, and verification spirals all stem from the multi-agent coordination overhead. The task itself was straightforward enough that one model could handle it entirely.

5. **HERMES showed near-zero variance** (1–2 min, 23–28k tokens). SLATE's variance was high (5–15 min, 428k–1.2M tokens), suggesting the subagent orchestration introduces unpredictable overhead.

6. **Test coverage was equivalent.** Despite the massive efficiency difference, both agents produced comparably thorough test suites. The quality of the final artifact was indistinguishable.

7. **Cost asymmetry.** SLATE averaged ~$1.72/run. HERMES token usage (~25k) would cost a fraction of a cent at typical API rates — roughly 100x cheaper per run.

---

## Appendix A: Task Prompt

The following prompt was given verbatim to both agents:

```
You are given a Rust project in the `starter_app/` directory. The project is a simple CLI task
manager. It does not compile, and it contains logic bugs.

Your job:

1. Read SPEC.md carefully — it defines the intended behavior of the application.
2. Fix all bugs so that `cargo build` succeeds with zero errors.
3. Correct all logic bugs so the application behaves as specified below.
4. Write a comprehensive unit test suite in `src/tests.rs` (added as `#[cfg(test)]` module or
   separate integration tests under `tests/`) covering all specified behaviors.
5. Ensure `cargo test` passes all tests with zero failures.
6. Ensure `cargo clippy` produces zero warnings.

Do not modify `Cargo.toml` dependencies. You may add `use` statements, new `impl` blocks, and
new files. Do not add external crates.

When you are done, write the following two files to your output directory:
- `token_usage.json` — `{ "input_tokens": N, "output_tokens": N, "tool_calls": N }`
- `edit_log.jsonl` — one line per file edit/create/delete:
  `{ "file": "src/foo.rs", "action": "edit", "ts": "<iso8601>" }`
```

---

## Appendix B: Application Spec (SPEC.md)

The spec defines the ground-truth behavior the fixed app must implement.

### Data Model

**`Priority` enum** — Three variants: `Low`, `Medium`, `High`
- `Display` formats as the variant name: `"Low"`, `"Medium"`, `"High"`
- `score()` returns: `High → 3`, `Medium → 2`, `Low → 1`

**`Task` struct**

| Field          | Type                   | Notes                                                   |
|----------------|------------------------|---------------------------------------------------------|
| `id`           | `usize`                | Unique, assigned by manager, starts at 1                |
| `title`        | `String`               | Non-empty string                                        |
| `priority`     | `Priority`             | One of the three variants                               |
| `completed`    | `bool`                 | Default `false`                                         |
| `created_at`   | `DateTime<Utc>`        | Set at creation time                                    |
| `completed_at` | `Option<DateTime<Utc>>`| `None` until `complete()` called; then `Some(Utc::now())`|

**`Task::complete()`** — Sets `completed = true` and `completed_at = Some(Utc::now())`

### TaskManager API

| Method                 | Signature                                          | Behavior                                          |
|------------------------|----------------------------------------------------|----------------------------------------------------|
| `new`                  | `() -> Self`                                       | Empty manager, `next_id = 1`                       |
| `add_task`             | `(&mut self, title: &str, priority: Priority) -> usize` | Adds task, returns id, increments `next_id`  |
| `complete_task`        | `(&mut self, id: usize) -> bool`                   | Marks complete, returns `true` if found            |
| `pending_count`        | `(&self) -> usize`                                 | Count where `completed == false`                   |
| `completed_count`      | `(&self) -> usize`                                 | Count where `completed == true`                    |
| `get_tasks_by_priority`| `(&self) -> Vec<&Task>`                            | All tasks sorted High → Low                        |
| `summary`              | `(&self) -> String`                                | Formatted summary (see below)                      |

### `summary()` Output Format

```
=== Task Summary (N pending, M completed) ===
[○] #1 [High] Fix production bug
[○] #3 [Medium] Write tests
[✓] #2 [Low] Buy groceries
```

- Header uses actual `pending_count()` and `completed_count()` values
- Tasks listed in priority order (High first)
- `[✓]` for completed, `[○]` for pending
- Format: `[STATUS] #ID [PRIORITY] TITLE`

---

## Appendix C: Starter App (Buggy Source Code)

The following files were given to both agents. Note: source files were intentionally placed in the
project root (`starter_app/`) rather than `starter_app/src/` — an additional structural issue both
agents had to detect and fix.

### `Cargo.toml`

```toml
[package]
name = "task_manager"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
chrono = { version = "0.4", features = ["serde"] }

[dev-dependencies]
```

### `lib.rs`

```rust
// lib.rs
// BUG: mod declarations are missing — task and manager modules are not exposed
// The agent must add:
//   pub mod task;
//   pub mod manager;
// and update main.rs to use them via `use task_manager::{manager::TaskManager, task::Priority};`
```

### `main.rs`

```rust
// main.rs - Task Manager CLI
// BUG: missing module declarations

fn main() {
    let mut manager = TaskManager::new();

    manager.add_task("Buy groceries", Priority::Low);
    manager.add_task("Fix production bug", Priority::High);
    manager.add_task("Write tests", Priority::Medium);

    // BUG: wrong method name called here
    manager.complete_tasks(1);

    println!("Pending tasks: {}", manager.pending_count());
    println!("Completed tasks: {}", manager.completed_count());

    let summary = manager.summary();
    println!("{}", summary);
}
```

### `task.rs`

```rust
// task.rs - Task and Priority definitions
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub enum Priority {
    Low,
    Medium,
    High,
}

impl Priority {
    // BUG: score values are inverted (High should be 3, Low should be 1)
    pub fn score(&self) -> u8 {
        match self {
            Priority::High => 1,
            Priority::Medium => 2,
            Priority::Low => 3,
        }
    }
}

// BUG: missing Display impl for Priority (needed by summary())

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Task {
    pub id: usize,
    pub title: String,
    pub priority: Priority,
    pub completed: bool,
    pub created_at: DateTime<Utc>,
    pub completed_at: Option<DateTime<Utc>>,
}

impl Task {
    pub fn new(id: usize, title: &str, priority: Priority) -> Self {
        Task {
            id,
            title: title.to_string(),
            priority,
            completed: false,
            created_at: Utc::now(),
            completed_at: None,
        }
    }

    pub fn complete(&mut self) {
        self.completed = true;
        // BUG: completed_at is never set
    }
}
```

### `manager.rs`

```rust
// manager.rs - TaskManager implementation
use crate::task::{Priority, Task};

pub struct TaskManager {
    tasks: Vec<Task>,
    next_id: usize,
}

impl TaskManager {
    pub fn new() -> Self {
        TaskManager {
            tasks: Vec::new(),
            next_id: 1,
        }
    }

    pub fn add_task(&mut self, title: &str, priority: Priority) -> usize {
        let id = self.next_id;
        self.tasks.push(Task::new(id, title, priority));
        // BUG: next_id is never incremented, so all tasks get id=1
        id
    }

    // BUG: method should be named complete_task (singular), main.rs calls complete_tasks
    pub fn complete_task(&mut self, id: usize) -> bool {
        if let Some(task) = self.tasks.iter_mut().find(|t| t.id == id) {
            task.complete();
            true
        } else {
            false
        }
    }

    pub fn pending_count(&self) -> usize {
        // BUG: logic is inverted — counts completed instead of pending
        self.tasks.iter().filter(|t| t.completed).count()
    }

    pub fn completed_count(&self) -> usize {
        self.tasks.iter().filter(|t| t.completed).count()
    }

    pub fn get_tasks_by_priority(&self) -> Vec<&Task> {
        let mut sorted: Vec<&Task> = self.tasks.iter().collect();
        // BUG: sort is ascending but score() is already inverted, so this accidentally
        // sorts Low→High instead of High→Low (double-bug that partially masks the score bug)
        sorted.sort_by_key(|t| t.priority.score());
        sorted
    }

    pub fn summary(&self) -> String {
        let tasks = self.get_tasks_by_priority();
        let mut lines = vec![
            format!(
                "=== Task Summary ({} pending, {} completed) ===",
                self.pending_count(),
                self.completed_count()
            )
        ];
        for task in tasks {
            let status = if task.completed { "✓" } else { "○" };
            // BUG: {} used for Priority but Display is not implemented
            lines.push(format!(
                "[{}] #{} [{}] {}",
                status, task.id, task.priority, task.title
            ));
        }
        lines.join("\n")
    }
}
```

---

## Appendix D: Known Bug Catalog

These 8 bugs were seeded into the starter app. Agents were NOT given this list.

| # | File         | Type           | Description                                                        |
|---|--------------|----------------|--------------------------------------------------------------------|
| 1 | `lib.rs`     | Compile error  | `pub mod task` and `pub mod manager` declarations missing          |
| 2 | `main.rs`    | Compile error  | Calls `complete_tasks(1)` but method is `complete_task` (singular) |
| 3 | `main.rs`    | Compile error  | `TaskManager` and `Priority` not imported                          |
| 4 | `task.rs`    | Logic bug      | `Priority::score()` inverted — `High=1, Low=3` should be `High=3, Low=1` |
| 5 | `task.rs`    | Compile error  | `Priority` has no `Display` impl but is used with `{}` in `summary()` |
| 6 | `task.rs`    | Logic bug      | `Task::complete()` does not set `completed_at`                     |
| 7 | `manager.rs` | Logic bug      | `add_task` never increments `next_id` — all tasks get `id = 1`    |
| 8 | `manager.rs` | Logic bug      | `pending_count()` counts completed tasks instead of pending ones   |

**Bonus bugs found by both agents:**
- `manager.rs`: `get_tasks_by_priority()` sorts ascending instead of descending (partially masked by the inverted `score()` values)
- `manager.rs`: Clippy `new_without_default` lint — both agents added `impl Default for TaskManager`
- File structure: Source files placed in project root instead of `src/` directory
