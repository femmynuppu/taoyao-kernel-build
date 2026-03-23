# PRD: KernelForge — Localhost AI Agent for Android Kernel Build Automation

**Version:** 1.0  
**Author:** feemi2  
**Target device:** Xiaomi 12 Lite (taoyao) / Xiaomi 13T (aristotle)  
**Stack:** ReSukiSU + SUSFS v2.1 + KPM  
**Status:** Draft

---

## 1. Problem Statement

Building a custom Android kernel with KernelSU + SUSFS + KPM patches into a CAF/QGKI source tree requires:

- Iterative error-fix-rebuild loops (this project has gone through 10+ iterations)
- Deep kernel engineering knowledge to identify root causes from noisy compiler logs
- Manual workflow editing on every error
- Context switching between GitHub Actions UI, log files, source code references, and patch tools

The entire debug loop (read log → identify error → research fix → edit yml → commit → wait 20–30 min → repeat) currently takes hours per session. Most of the work is mechanical: pattern matching on compiler errors and applying known fix classes.

**Core insight:** A local AI agent can close this loop autonomously.

---

## 2. Goals

| Goal | Priority |
|------|----------|
| Reduce build-to-working-kernel iteration time from hours to minutes | P0 |
| Eliminate manual log analysis and fix authoring | P0 |
| Support full taoyao + aristotle kernel targets | P0 |
| Run completely offline / localhost (no cloud dependency for core loop) | P1 |
| Modular — easy to add new device targets | P1 |
| Produce reproducible, auditable patch history | P2 |

## Non-Goals

- Not a cloud service — runs on user's local machine only
- Not a general-purpose kernel CI system
- Does not auto-flash to device (safety boundary)
- Does not modify upstream SUSFS/ReSukiSU source trees

---

## 3. Users

**Primary user:** Femmy (feemi2) — Android kernel developer, non-GKI device maintainer, solo operator.

**Persona:** Highly technical, builds for personal use + ~1000 module users. Wants results fast, understands the stack, but doesn't want to debug every CAF quirk manually. Comfortable with CLI, git, Python.

---

## 4. System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    KernelForge Agent                    │
│                                                         │
│  ┌──────────┐    ┌───────────┐    ┌──────────────────┐  │
│  │  Trigger │───▶│   Build   │───▶│  Log Collector   │  │
│  │  Engine  │    │  Runner   │    │  (local / GHA)   │  │
│  └──────────┘    └───────────┘    └────────┬─────────┘  │
│                                            │             │
│                                   ┌────────▼─────────┐  │
│                                   │  Error Analyzer  │  │
│                                   │  (LLM + rules)   │  │
│                                   └────────┬─────────┘  │
│                                            │             │
│                                   ┌────────▼─────────┐  │
│                                   │  Patch Generator │  │
│                                   │  (deterministic) │  │
│                                   └────────┬─────────┘  │
│                                            │             │
│                                   ┌────────▼─────────┐  │
│                                   │  Patch Applier   │  │
│                                   │  + Commit + Push │  │
│                                   └────────┬─────────┘  │
│                                            │             │
│                       ◀────── loop ────────┘             │
│                       (until success or max_retries)     │
└─────────────────────────────────────────────────────────┘
```

---

## 5. Core Modules

### 5.1 Trigger Engine

**Responsibility:** Start a build run, either manually or on schedule.

```python
# Interface
class TriggerEngine:
    def trigger_local(self, target: KernelTarget) -> BuildRun
    def trigger_gha(self, repo: str, workflow: str, inputs: dict) -> BuildRun
    def watch_gha_run(self, run_id: str) -> BuildStatus
```

**Inputs:**
- Target device config (taoyao / aristotle)
- Feature flags (KPM, SUSFS, BBR, ZRAM)
- Trigger type: `local` | `github_actions`

### 5.2 Build Runner (Local Mode)

Runs the build locally using Docker to replicate the GitHub Actions environment exactly.

```
Docker image: ubuntu:22.04 (matches GHA runner)
Volumes:
  - kernel_source/ → /workspace/kernel
  - toolchains/    → /workspace/toolchains (cached)
  - patches/       → /workspace/patches
```

**Why Docker:** Eliminates GLIBC mismatch issues (the exact bug that caused clang failures in this project).

### 5.3 Log Collector

Streams or pulls build logs. Normalizes output regardless of source (local make output vs GHA artifact).

```python
class LogCollector:
    def from_make_stdout(self, stream) -> BuildLog
    def from_gha_artifact(self, zip_path: str) -> BuildLog
    def from_gha_api(self, run_id: str) -> BuildLog

class BuildLog:
    errors: list[CompilerError]
    warnings: list[str]
    raw: str
    success: bool
    failed_target: str | None  # e.g. "arch/arm64" or "fs"
```

### 5.4 Error Analyzer

**The intelligence layer.** Takes a `BuildLog` and produces structured `ErrorRecord` list.

**Two-tier analysis:**

**Tier 1 — Rule Engine (fast, deterministic, 90% of cases):**

```python
# Pattern database for known Android/CAF kernel errors
KNOWN_PATTERNS = [
    ErrorPattern(
        id="VOID_RETURN_VALUE",
        regex=r"error: void function '(\w+)' should not return a value",
        files=["include/linux/qcom_scm.h"],
        fix_class=FixClass.REMOVE_RETURN_FROM_VOID,
        risk=Risk.LOW,
    ),
    ErrorPattern(
        id="UNDECLARED_SYMBOL",
        regex=r"error: use of undeclared identifier '(\w+)'",
        fix_class=FixClass.ADD_DEFINE_OR_HEADER,
        risk=Risk.MEDIUM,
    ),
    ErrorPattern(
        id="INCOMPLETE_TYPE",
        regex=r"error: incomplete type '(\w+)' where a complete type is required",
        fix_class=FixClass.ADD_FORWARD_DECL,
        risk=Risk.MEDIUM,
    ),
    ErrorPattern(
        id="ARITY_MISMATCH",
        regex=r"error: too (few|many) arguments to function '(\w+)'",
        fix_class=FixClass.FIX_FUNCTION_ARITY,
        risk=Risk.LOW,
    ),
    ErrorPattern(
        id="MISSING_HEADER",
        regex=r"error: '(.+\.h)' file not found",
        fix_class=FixClass.COPY_MISSING_HEADER,
        risk=Risk.MEDIUM,
    ),
    ErrorPattern(
        id="KSU_TP_HOOKS_NON_GKI",
        regex=r"TP hooks are incompatible with Non-GKI",
        fix_class=FixClass.REMOVE_ERROR_LINE_IN_KBUILD,
        risk=Risk.LOW,
    ),
    ErrorPattern(
        id="IMPLICIT_FUNCTION_DECL",
        regex=r"error: implicit declaration of function '(\w+)'",
        fix_class=FixClass.ADD_STUB_OR_REMOVE_CALL,
        risk=Risk.HIGH,  # Requires understanding call context
    ),
]
```

**Tier 2 — LLM Analysis (for unknown errors):**

When no rule matches, calls local LLM (via Ollama / Claude API) with:

```python
ANALYSIS_PROMPT = """
You are a senior Android kernel engineer (CAF/QGKI, 5.4).
Analyze this build error and return JSON:
{
  "error_class": "<class>",
  "root_cause": "<1-sentence explanation>",
  "affected_file": "<path>",
  "fix_strategy": "<what to change>",
  "risk": "low|medium|high",
  "needs_human": true|false
}

Build error:
{error_context}
"""
```

### 5.5 Patch Generator

Takes `ErrorRecord` → generates precise file patches.

**Key constraint: MINIMAL changes only. No blind substitution.**

```python
class PatchGenerator:
    def generate(self, error: ErrorRecord, kernel_dir: Path) -> Patch

class Patch:
    file: Path
    strategy: PatchStrategy  # sed | perl | python_ast | manual
    diff: str                # unified diff for audit log
    description: str
    reversible: bool
    
    def apply(self) -> bool
    def revert(self) -> bool
```

**Fix strategies by error class:**

```
VOID_RETURN_VALUE:
  Strategy: PERL regex, only match explicit non-void type prefix
  Pattern: match type token BEFORE function name, not after
  Safe: yes — only affects inline stubs in header

UNDECLARED_SYMBOL (SUSFS):
  Strategy: check SUSFS repo for canonical define
  If not found: create typed stub header with documented bit value
  Safe: yes if bit allocation is verified against kernel i_state usage

ARITY_MISMATCH:
  Strategy: count args in declaration vs call site
  Add zero-value extra args at end (write-only callers)
  Safe: yes for n_rd=0 pattern

KSU_TP_HOOKS_NON_GKI:
  Strategy: sed delete lines containing $(error ...) in Kbuild
  Safe: yes — non-GKI kernel cannot use TP hooks by design

IMPLICIT_FUNCTION_DECL (HIGH risk):
  Strategy: LLM + human confirm before applying
  Safe: blocked until user approves
```

### 5.6 Patch Applier + Commit Loop

```python
class PatchLoop:
    max_iterations: int = 15
    
    def run(self, target: KernelTarget) -> LoopResult:
        for i in range(self.max_iterations):
            result = self.build()
            if result.success:
                return LoopResult.SUCCESS
            
            errors = self.analyzer.analyze(result.log)
            
            # Block on high-risk errors — require human
            high_risk = [e for e in errors if e.risk == Risk.HIGH]
            if high_risk:
                self.notify_human(high_risk)
                return LoopResult.NEEDS_HUMAN
            
            patches = [self.generator.generate(e) for e in errors]
            
            # Dry-run first
            for p in patches:
                if not p.dry_run():
                    self.log(f"Patch dry-run failed: {p.description}")
                    return LoopResult.PATCH_FAILED
            
            # Apply + record
            for p in patches:
                p.apply()
                self.audit_log.append(p)
            
            self.commit(f"fix({i}): {[p.description for p in patches]}")
        
        return LoopResult.MAX_RETRIES
```

---

## 6. Device Targets Config

```yaml
# config/targets.yaml

taoyao:
  display_name: "Xiaomi 12 Lite"
  kernel_version: "5.4.289-qgki"
  kmi: "android12-5.10"  # note: QGKI 1.0
  source:
    repo: "https://github.com/richardqcarvalho/kernel_xiaomi_taoyao"
    branch: "bka"
  defconfig:
    base: "gki_defconfig"
    fragments:
      - "vendor/lahaina_QGKI.config"
      - "vendor/taoyao_QGKI.config"
  toolchain:
    name: "proton-clang"
    url: "https://github.com/kdrag0n/proton-clang"
    host_requires: "ubuntu-22.04"  # GLIBC 2.35 compat
  ksu:
    variant: "ReSukiSU"
    mode: "builtin"  # no LKM on 5.4
    branch: "builtin"
  features:
    susfs: true
    kpm: true
    bbr: true
    zram: true

aristotle:
  display_name: "Xiaomi 13T"
  kernel_version: "5.10.198-android12"
  kmi: "android12-9"
  source:
    repo: "https://github.com/MiCode/Xiaomi_Kernel_OpenSource"
    branch: "bsp-aristotle-s-oss"
  defconfig:
    base: "gki_defconfig"  # GKI 2.0
  toolchain:
    name: "neutron-clang"
    url: "https://github.com/Neutron-Toolchains/clang-build-catalogue"
    host_requires: "ubuntu-24.04"
  ksu:
    variant: "SukiSU-Ultra"
    mode: "lkm"  # GKI 2.0 supports LKM
    branch: "susfs-main"
  features:
    susfs: true
    kpm: true
    bbr: true
    zram: true
```

---

## 7. Build Pipeline — Full Flow

```
User: kernelforge build taoyao
         │
         ▼
1. Load target config (taoyao)
         │
         ▼
2. Spin up Docker container (ubuntu:22.04)
   - Mount kernel_source/
   - Mount toolchain cache
         │
         ▼
3. Clone / update kernel source
   - git fetch --depth=1 origin bka
   - Check KernelSU-Next submodule → remove
         │
         ▼
4. Integrate ReSukiSU
   - Run setup.sh builtin
         │
         ▼
5. Integrate SUSFS
   - Clone kernel-5.4 branch
   - Copy susfs.c, susfs.h, susfs_def.h
   - Apply 50_add_susfs_in_kernel-5.4.patch
   - Log hunk failures for analysis
         │
         ▼
6. Apply known pre-build patches
   - qcom_scm.h: remove TP hook error, fix void/non-void returns
   - rpmb-mtk.c: fix undeclared mtk_mmc_host
   - coresight: fix usb_qdss_alloc_req arity
         │
         ▼
7. Generate defconfig
   - gki_defconfig → lahaina_QGKI.config → taoyao_QGKI.config
   - Append KSU + SUSFS + KPM + BBR + ZRAM config
   - olddefconfig
         │
         ▼
8. BUILD LOOP (max 15 iterations)
   │
   ├── make -j$(nproc) 2>&1 | tee build.log
   │
   ├── Parse build.log → extract errors
   │
   ├── For each error:
   │   ├── Rule match? → generate patch → apply
   │   └── No match? → LLM analysis → low/medium: apply | high: pause
   │
   ├── Commit fixes: "fix(N): <description>"
   │
   └── Repeat until success
         │
         ▼
9. Package AnyKernel3 zip
         │
         ▼
10. Output:
    - ReSukiSU_taoyao_YYYYMMDD_HHMM.zip (flash via TWRP)
    - patch_history.json (audit log)
    - build_summary.md
```

---

## 8. CLI Interface

```bash
# Build target
kernelforge build taoyao
kernelforge build aristotle --no-kpm

# Analyze existing log
kernelforge analyze build.log

# Check patch history
kernelforge log taoyao

# Interactive mode (ask before applying each patch)
kernelforge build taoyao --interactive

# Dry run (generate patches, don't apply)
kernelforge build taoyao --dry-run

# Reset to clean state
kernelforge reset taoyao
```

---

## 9. Tech Stack

| Component | Technology | Reason |
|-----------|------------|--------|
| Runtime | Python 3.11+ | Fast iteration, good subprocess/file handling |
| LLM (local) | Ollama + codellama:13b | Offline, fast on modern hardware |
| LLM (cloud fallback) | Claude API (claude-sonnet-4-6) | Higher accuracy for complex errors |
| Build isolation | Docker (ubuntu:22.04 / 24.04) | GLIBC version control |
| GHA integration | GitHub CLI (`gh`) | Trigger + artifact download |
| Config | YAML + Pydantic | Validated, typed |
| Patch format | Unified diff | Auditable, reversible |
| Audit log | SQLite | Query patch history |

---

## 10. Risk Map

### Build Risks (kernel-specific)

| Risk | Severity | Mitigation |
|------|----------|------------|
| Void function gets `return 0` (our bug) | MEDIUM | Explicit type-list matching, not regex lookahead |
| SUSFS partial hunk failure silently breaks symbol | HIGH | Verify each patched symbol compiles, check include chain |
| qcom_scm.h over-patching breaks ARM MM fault handler | HIGH | Dry-run before apply, build arm64/mm/ first as smoke test |
| KPM config conflict with MODULE_SIG | MEDIUM | Explicit `CONFIG_MODULE_SIG=n` in append step |
| rpmb-mtk NULL dereference at runtime | LOW | NULL check exists in rpmb driver — NULL mmc = graceful skip |
| Bootloop after flash | HIGH | Always backup boot.img; AnyKernel3 includes fallback |

### Agent Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| Agent applies wrong fix to wrong file | HIGH | Dry-run always, unified diff shown before apply |
| LLM hallucinates fix for unknown error | HIGH | High-risk errors require human approval |
| Max retries hit without progress | MEDIUM | Detect loop (same error twice) → escalate to human |
| Patch corrupts kernel source | HIGH | git stash before each iteration, auto-revert on parse fail |

---

## 11. Safety Constraints (Non-Negotiable)

```python
SAFETY_RULES = [
    # Never modify core security files without explicit flag
    "NEVER auto-patch: security/selinux/*, creds.c, capability.c",
    
    # Never apply untested fix to boot-critical path
    "NEVER auto-patch: arch/arm64/kernel/head.S, init/main.c",
    
    # Require human confirmation for high-risk error class
    "BLOCK_ON_RISK: implicit_function_declaration in fs/, mm/",
    
    # Always preserve original before patching
    "ALWAYS: git stash / backup before patching kernel source",
    
    # Never auto-flash — output is always a flashable zip, not flashed
    "NEVER: auto-flash to device",
    
    # Patch history is immutable
    "ALWAYS: append-only audit log for all applied patches",
]
```

---

## 12. Project Structure

```
kernelforge/
├── main.py                    # CLI entry point
├── config/
│   ├── targets.yaml           # Device target definitions
│   └── error_patterns.yaml    # Rule engine patterns
├── core/
│   ├── trigger.py             # Build trigger (local/GHA)
│   ├── runner.py              # Docker-based local build
│   ├── log_collector.py       # Log parsing + normalization
│   ├── analyzer.py            # Error analysis (rule + LLM)
│   ├── generator.py           # Patch generation
│   ├── applier.py             # Patch application + git
│   └── loop.py                # Main iteration loop
├── patches/
│   ├── taoyao/                # Device-specific pre-patches
│   │   ├── qcom_scm_fix.py
│   │   ├── susfs_def_stub.py
│   │   └── rpmb_mtk_fix.py
│   └── common/                # Cross-device patches
│       └── ksu_tp_hooks.py
├── db/
│   └── patches.db             # SQLite audit log
├── docker/
│   ├── Dockerfile.ubuntu2204  # taoyao build env
│   └── Dockerfile.ubuntu2404  # aristotle build env
└── tests/
    ├── test_analyzer.py
    └── fixtures/
        └── build_logs/        # Real build log samples for testing
```

---

## 13. Phase 1 MVP Scope

**Milestone 1 (Week 1–2):** Core loop working for taoyao
- [ ] Log parser handles all 10 build log variants from this project
- [ ] Rule engine covers: VOID_RETURN, UNDECLARED_SYMBOL, ARITY_MISMATCH, TP_HOOKS_NON_GKI
- [ ] Docker build environment matches GHA ubuntu-22.04
- [ ] Patch applier + git commit working

**Milestone 2 (Week 3):** LLM integration
- [ ] Ollama integration for offline analysis
- [ ] Claude API fallback for complex errors
- [ ] Human-in-loop for HIGH risk errors

**Milestone 3 (Week 4):** Aristotle + polish
- [ ] aristotle target support (LKM mode, ubuntu-24.04)
- [ ] CLI interface complete
- [ ] Audit log (SQLite)
- [ ] README + usage docs

---

## 14. Success Metrics

| Metric | Target |
|--------|--------|
| Taoyao: iterations to successful build | ≤ 3 (currently 10+) |
| Build-to-kernel time (local, cached toolchain) | < 45 min |
| Known error class coverage (rule engine) | ≥ 85% of observed errors |
| False patch rate (wrong fix applied) | 0% (dry-run gate) |
| Human interruptions per build | < 1 on known targets |

---

## 15. Current Error State (as of 2026-03-23 build-log-1215)

Two active errors blocking taoyao build:

**Error 1 — qcom_scm.h void functions (fixed in v10 workflow):**
```
include/linux/qcom_scm.h:235: error: void function 'qcom_scm_cpu_power_down'
should not return a value
```
Root cause: our v9 perl fix matched void functions. v10 uses explicit type list.

**Error 2 — INODE_STATE_SUS_KSTAT undeclared (fixed in v10 workflow):**
```
fs/proc/task_mmu.c:372: error: use of undeclared identifier 'INODE_STATE_SUS_KSTAT'
```
Root cause: susfs_def.h stub was empty. v10 creates proper stub with:
`#define INODE_STATE_SUS_KSTAT (1 << 31)` — bit 31 of `inode->i_state` (reserved).

Both fixes are in `build-taoyao-v10.yml`.

---

*KernelForge PRD v1.0 — feemi2 / 2026-03-23*
