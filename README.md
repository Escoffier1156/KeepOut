```
  🔒 _  _____ _____ _____   ____  _   _ _____ 
  | |/ / ____| ____|  _ \ / __ \| | | |_   _|
  | ' /|  _| |  _| | |_) | |  | | | | | | |  
  | . \| |___| |___|  __/| |__| | |_| | | |  
  |_|\_\_____|_____|_|    \____/ \___/  |_|  
```

# 🔒 KeepOut

> **Production-ready, lightweight code protection powered by Smart Syntax Locking (AST Alpha-Renaming) & Deterministic Property-Based Testing (PBT).**

**KeepOut** is a lightweight CLI tool designed to physically protect critical code sections from external changes—whether caused by future oversights, unintended edits by teammates, or automated AI refactorings—by marking regions with comment directives (`# [LOCK]` ... `# [/LOCK]`).

Refactored for maximum production usability, KeepOut features **Smart Syntax Locking** ($O(1)$ AST Structural Hash & Variable Alpha-Renaming) and **Deterministic Property-Based Testing (10,000+ Reproducible Fuzzing Samples + Counterexample Caching)**, ensuring instant verification, zero flaky CI builds, and zero freezes across large-scale repositories.

---

## 🌟 2 Core Production Engines

| Engine | Syntax Directive | How It Works | Performance & Reliability | Ideal Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **Smart Syntax Lock** *(Default)* | `# [LOCK]` or `# [LOCK: ast]` | **AST Alpha-Renaming & Structural Hash**. Ignores comments, whitespace, formatting, indentation, and variable renames (`x` ➔ `val`). | ⚡ $O(1)$ (<1ms)<br>Zero freezes | Formatting & linting-heavy codebases where code structure must stay intact. |
| **Deterministic PBT Sub-Engine** | `# [LOCK: pbt]` or `# [LOCK: logic]` | **Deterministic Property-Based Testing (10,000 Samples)** + **Counterexample Auto-Caching**. Reproducible fixed seed (`seed=42`). | 🎯 Milliseconds<br>Zero Flaky CI Runs | Core algorithms, AI refactorings, functions with complex loops or external libraries. |
| **Strict Lock** | `# [LOCK: strict]` | SHA-256 exact byte hash matching. Rejects any edit including spaces or comments. | ⚡ $O(1)$ Instantaneous | Security constants, preventing re-occurrence of critical bugs. |

---

## ⚠️ Lock Invariants, Not Implementations

This is the one rule that decides whether KeepOut helps you or hurts you.

KeepOut enforces *"this must not change."* It cannot tell you *"this is correct."*
Lock the wrong thing and you have built a machine for preserving a bug.

**A real case.** A production trading engine locked nine regions with
`# [LOCK: logic]`. Every one of them was an *implementation*:

| Locked region | What it actually contained |
| :--- | :--- |
| `KabutanScraper` | a parser reading the trade-count column as volume |
| `LineNotifier` | notifications that failed 403 in silence |
| `ExtremeValueTheoryEVT` | `md5(ticker) % 13` dressed up as tail-risk analysis |
| `MonteCarloPathSimulator` | the same, as path simulation |
| `JQuantsAPIClient` | a hardcoded list of fictional stocks at fixed prices |
| `ATRDynamicTrailingEngine` | a `+4%` hard cut, and targets of `+58%` on a day trade |
| `SlotRotationManager` | a 5-slot book that grew to 6 positions |

KeepOut protected all of it, faithfully and exactly as instructed, while the
system quietly reported profits it had not made. The tool did its job. The
choice of what to lock did not.

### The test

Ask one question about the region:

> **If this changed silently, would every number downstream become quietly wrong?**

| Lock it ✅ | Leave it open ❌ |
| :--- | :--- |
| Physical/financial constants (fee rates, unit sizes) | Anything that parses an external site or API |
| The single definition of a core calculation | Tunable strategy parameters |
| Invariant assertions - the guarantees themselves | Logic you intend to evolve on evidence |
| Security constants, protocol magic numbers | Anything you may need to hotfix under time pressure |

The right-hand column matters as much as the left. Locking an HTML scraper is
actively dangerous: when the site changes its markup mid-session, the safety
property you need is *being able to patch it in three minutes* - and the lock
works directly against that.

### Locking the same engine, done well

The same project, rebuilt, locks four regions and nothing else:

```python
# [LOCK: strict]
FRICTION_PER_SIDE = 0.00125   # duplicated in the web UI; drift = silent disagreement
# [/LOCK]

# [LOCK: pbt]
def net_pnl(entry_price: float, exit_price: float, shares: int):
    """Restructure freely - the numbers must not move."""
# [/LOCK]

# [LOCK]
def assert_invariants(self) -> None:
    """The 5-slot guarantee itself."""
# [/LOCK]
```

Its scrapers, thresholds and selection logic are deliberately left open.

---

## 🧠 Deterministic PBT & Counterexample Caching

```
[ Original Code ]  ──┐
                     ├──> [ AST Alpha-Renaming Engine ] ──> Match? ➔ PASS ✨ (<0.001s)
[ Modified Code ]  ──┘                   │
 (Variable renames,                      └── Mismatch? (AST Structure Altered)
  comments, formatting                             │
  ignored automatically)                           ▼
                                     [ 1. Check Cached Counterexamples ] (keepout.json)
                                                   │
                                                   ├── Fail? ➔ Instant Regression Block 🚨 (0.0001s)
                                                   └── Pass? ➔ [ 2. Deterministic PBT Engine ]
                                                               (10,000 Reproducible Inputs, seed=42)
                                                                       │
                                                                       ├── Output Match ➔ PASS ✨
                                                                       └── Discrepancy ➔ BLOCK 🚨
                                                                           (Auto-cache Counterexample)
```

1. **Deterministic Seed (No Flaky Tests)**: Fixed PRNG seed guarantees 100% reproducible test vectors across local and CI runs.
2. **Counterexample Caching**: When a bug or logic mismatch is caught, KeepOut automatically caches the failing input parameter inside `keepout.json`. On subsequent checks, KeepOut checks cached counterexamples **first**, preventing regressions in 0.0001 seconds.

### What `[LOCK: pbt]` can and cannot verify

The PBT engine executes the region and compares outputs, so it applies to a
specific shape of code:

| Works | Does not work |
| :--- | :--- |
| Pure functions of any arity | Functions performing I/O (network, disk, clock) |
| Parameters annotated `int` / `float` / `bool` | Parameters taking objects, dicts or collections |
| Deterministic outputs | Anything whose result depends on external state |

Argument domains follow the type annotations - annotate your parameters and the
engine drives them properly. For code PBT cannot execute, use `[LOCK]` (AST
structural) or `[LOCK: strict]` instead; both verify without running anything.

**If no sample executes, KeepOut blocks rather than passes.** A lock reporting
success while having verified nothing is worse than no lock at all, so this case
returns `pbt_unverifiable` and names the arity it detected.

---

## 🐳 1. Running with Docker (Recommended)

No local setup required.

```bash
# Build Docker image
docker build -t keepout .

# Alias for native CLI execution
alias keepout="docker run --rm -v \$(pwd):/workspace keepout"

# Usage
keepout init .
keepout check .
```

---

## 💻 2. Local Installation & Usage (Pip / Pixi)

```bash
# Install via pip
pip install -e .

# Or using Pixi
pixi run pip install -e .
```

### CLI Commands

```bash
# Initialize locks in keepout.json
keepout init

# Verify lock integrity
keepout check

# Install Git Pre-commit hook
keepout install-hook
```

---

## 📜 Directive Syntax Protocol

### 🌿 Smart Syntax Lock (Default: Structure Fixed, Formatting & Renames Ignored)
```python
# [LOCK: ast name="calc"]
def calculate_product(input_value):
    # Variable renames, comments, and formatting edits are allowed
    result_value = input_value * 4
    return result_value
# [/LOCK]
```

### 🧠 Deterministic Property-Based Testing Lock (10,000 Reproducible Samples)
```mojo
# [LOCK: pbt name="fast_calc"]
fn fast_calc(x: Int) -> Int:
    return x * 4
# [/LOCK]
```

### 🔒 Strict Lock (Exact Byte Freeze)
```c
// [LOCK: strict]
const int MAX_RETRY_COUNT = 5;
// [/LOCK]
```

---

## 🎯 Choosing a Mode

| Mode | Verifies | Reach for it when |
| :--- | :--- | :--- |
| `[LOCK: strict]` | exact bytes | constants, magic numbers - one correct value exists |
| `[LOCK]` / `[LOCK: ast]` | code structure | guarantees and algorithms that may be reformatted or renamed, never restructured |
| `[LOCK: pbt]` | observable behaviour | pure calculations you *do* want refactored, as long as outputs never move |

## 🧪 Running Tests

```bash
pytest
```

---

## 📄 License

MIT License
