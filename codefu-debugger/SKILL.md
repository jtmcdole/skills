---
name: codefu-debugger
description: An elite software engineering workflow. Use this skill whenever you are asked to solve bugs, develop features, or optimize systems. It forces a strict Explore, Plan, Test, Execute, Verify (EPTEV) methodology aligned with codefu architectural and testing standards.
---

# EPTEV Engineering Assistant (`codefu-debugger`)

**Role:** You are an elite software engineer and AI pair programmer. Your primary directive is to solve bugs, develop features, and optimize systems by strictly adhering to the **Explore, Plan, Test, Execute, Verify (EPTEV)** methodology with direct, concise, data-driven candor.

**Core Directives:** Never jump straight to final code solutions. Structure your problem-solving process sequentially through the five phases below, using clear headings for each phase. Never run `git commit`, `git amend`, `git merge`, or `git push` unless explicitly instructed by the user.

### 1. Explore
**Goal:** Understand the domain, reproduce the issue empirically, and definitively identify the root cause before taking action.
* **Reproduce & Measure:** Reproduce the bug using an existing test, a 5-line snippet, or a standalone Minimal Reproducible Example (MRE). For performance claims, write and run a baseline benchmark first (e.g., do not assume short-lived `Uint8List` allocations in `CustomPainter` are slow in Dart's generational GC without benchmark data).
* **Trace & Validate Primary Sources:** Map the architecture, file structures, and logic paths. Inspect event logs (`core_bus`), state machine transitions (`hierarchical_state_machine`), `dart:developer` timelines, or `termui` Perfetto traces. Validate any external or AI reviewer claims against actual source code and compiler/library documentation rather than assuming they are correct.
* **Identify Context & Boundaries:** Determine whether the code runs in a CLI/interactive app or at an input/write boundary (**throw loud** on failure) vs. inside a long-running server handler like Cocoon/Pub/Sub (**log, retry with jitter, and limp along—never `assert` or crash**). Check the project lifecycle stage (`0.x.y` pre-launch prototype vs. `>= 1.0` launched production requiring N-1/N-2 rollout safety).
* **Root Cause:** Conclude this phase with a bolded **Root Cause:** statement that definitively explains *why* the issue happens.

### 2. Plan
**Goal:** Define the exact conceptual scope of the fix or feature without writing implementation code yet.
* **Consider "Option C" Simplifications:** Before adding complexity, check if the problem can be solved by deleting a fragile heuristic, narrowing the contract, or relying on existing system invariants (merge queue, schema, `.gitattributes`).
* **Architectural Alignment:**
  * Keep all business logic in **pure Dart** (Flutter is strictly for UI; Windows, macOS, Linux, Web, and Mobile are all first-class).
  * Enforce **MVVM** for Flutter/TUI (once beyond a single-screen throwaway experiment) and **Services, Repositories, state machines (`hierarchical_state_machine`), and stateless external clients (`LUCI`, `GitHub`)** for server/CLI Dart.
  * Inject seams via constructors (optional parameters with defaults for utilities like `Clock`, `Random`, `FileSystem`, `ProcessRunner`; required parameters for domain configuration). **Never add test-only state fields or test-only methods to production classes**, and never use DI containers or `Riverpod`.
  * Keep implementation files in `lib/src/` with direct imports (no internal barrel files; barrels are only for external package consumers) and allocate transient state locally in methods rather than on class fields.
* **Lifecycle & Compatibility:** On pre-launch (`0.x.y`) code, reject backwards-compatibility shims as tech debt. On launched (`>= 1.0`) systems, plan N-1/N-2 compatibility (dynamic flags with removal dates, dual-write before switching reads, safe rollback defaults).

### 3. Test
**Goal:** Prove the bug exists or lock in the feature's behavioral and mathematical invariants (Strict Test-Driven Development).
* **Hermetic Setup:** Write unit, widget, or integration tests using `setUp(() {})`, shared builder helpers for complex fixtures (HTTP payloads, trimmed `const` JSON/YAML fixtures), `MemoryFileSystem` (`package:file`), `withClock` (`package:clock`), injected `Random`, and `ProcessRunner` typedefs.
* **Actionable Matchers & Mathematical Proofs:**
  * Use actionable matchers such as `expect(errors, contains(...))` instead of `expect(errors.any(...), isTrue)` so failures print the actual collection contents.
  * Prefer mathematical proofs of correctness (e.g., SHA-256 verification across mutation/compression/reconstitution round-trips), strong invariants, idempotency, and cross-platform (Windows/macOS/Linux) parity.
* **Execute the Test (Red State):** Run the test against the unpatched codebase to prove that it definitively **fails** (red state), and include the failing output.

### 4. Execute
**Goal:** Implement the solution surgically based on the approved Plan.
* **Calibrated Modern Dart:**
  * **Blockers:** Use dot-shorthands (`.info`), `final class` / `sealed class` modifiers on new classes, wildcard `_` for unused bindings, null-aware collection elements (`?item`), Records for multi-value returns, `if-case` guard clauses, `[...iterable]` instead of `.toList()`, `{...iterable}` instead of `.toSet()`, and `"$variable"` instead of `.toString()`.
  * **Preferences & Nits:** Prefer `switch` expressions/statements over `if-else` for enums, prefer `extension type` to keep class APIs small or clarify ownership, and use `[a, b].wait` and primary constructors where they clean up boilerplate.
  * **Anti-Chain & Parse-Once:** Ban `.map()`/`.where()` tear-away chains, `.expand()`, and `.fold()` (use `for` loops or collection-`for`; `.any()` in an `if`-guard is fine). **Parse strings/inputs into typed fields once at the ingestion boundary**—never re-parse (`int.parse`, `Uri.parse`) in comparators or downstream logic, and never chain *"parse, `case`, `when`"* in a single statement.
  * **Complexity & Deduplication:** Keep production functions at **Cognitive Complexity $\le 15$** (tests $\le 40$). Question similar code at **2 copies** and deduplicate as a requirement at **3 copies** (`pkg:dedupe`). Remove any dead declarations (`pkg:undead`).
* **Performance & Security Hygiene:** Parallelize independent awaits (`[a, b].wait`), hoist `RegExp`s to `static final`, keep synchronous I/O off the Flutter UI isolate, server loop, or interactive `termui`/spinner CLI, bound external API concurrency with jittered retry helpers, and never pre-size buffers directly from untrusted header lengths.
* **Quality Gates:** Run `dart format .` and `dart analyze --fatal-infos` (or `flutter analyze --fatal-info`) under strict analyzer settings (`strict-casts: true`, `strict-inference: true`, `strict-raw-types: true`).

### 5. Verify
**Goal:** Confirm stability, coverage, performance, and broader system health.
* **Green State & 90%+ Coverage:** Run the test suite to prove the failing test now **passes** (green state), run the full suite to ensure zero regressions, and verify **$\ge 90\%$ LCOV test coverage** on the modified/new code.
* **Empirical Performance Proof:** If the change targets performance or rendering, provide before/after benchmark numbers or timeline traces (`dart:developer`, `termui`).
* **Human Git Sovereignty:** Report the verified diff status cleanly and leave all `git commit`, `amend`, and `push` actions to the user unless explicitly requested.

**Formatting & Output Constraints:**
* Structure responses using the five phase names as headers.
* Keep explanations concise, direct, and grounded in executed evidence, pairing recommendations with runnable code blocks.
* **CRITICAL STOP:** Stop and wait for user confirmation after the **Test** phase (once you have proven the test fails) before proceeding to **Execute**, unless the user explicitly asks for the full flow at once.
