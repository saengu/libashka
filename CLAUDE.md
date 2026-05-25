# Project Overview
This document serves as the implementation blueprint and system prompt for converting Babashka into an embeddable, shared dynamic library (`libashka`) using GraalVM Native Image. 

## System Architecture & Boundaries
The target architecture utilizes GraalVM Isolates to manage independent Babashka runtime instances inside a host process (C, Zig, Rust, Swift).
Need to support Linux (`x86_64`, `aarm64` )

### Supported Deployment Matrix
*   **Linux**: `x86_64`, `arm64` (aarch64) $\rightarrow$ Outputs `libashka.so`
*   **macOS**: `x86_64`, `arm64` (M1/M2/M3) $\rightarrow$ Outputs `libashka.dylib`
*   **Windows**: `x86_64` $\rightarrow$ Outputs `libashka.dll` & `libashka.lib`


## Key Technical Pillars
*   **GraalVM Native Image C API**: Uses `org.graalvm.nativeimage.c.function.CEntryPoint` to export standard C linkage functions.
*   **Isolate Management**: Every execution context requires a pointer to a GraalVM isolate thread (`graal_isolatet_t*`).
*   **Cross-Platform Calling Conventions**: Uses explicit platform macros (`__declspec(dllexport)` on Windows) to handle dynamic symbol exports cleanly.
*   **Data Serialization Layer**: Input parameters, output results, and callback payloads are passed across the FFI boundary as **JSON strings** to eliminate complex structure mapping.
*   **Bi-directional Callbacks**: Babashka executes host functions by resolving C function pointers registered in a global or context-specific SCI registry.
*   **Pod Compatibility**: Babashka pods communicate over standard I/O streams or loops. `libashka` must preserve standard `babashka.pods` namespaces and ensure background sub-processes spin up correctly.

## 5. Instructions for Assistant Code Generation
1. Always parse and serialize incoming payloads using a tiny JSON engine (`cheshire.core`) directly at the FFI border.
2. Store separate SCI states inside a Thread-safe atomic lookup Map indexed by sequential integers (`bb_context_id`).
3. Ensure error logs and stacks are caught on the Clojure end and passed outwards gracefully as a JSON payload schema: `{"error": "Stacktrace detail..."}` to prevent memory exceptions inside host systems.
4. Handle Windows specific platform process isolation features smoothly. Ensure your custom Native Image configuration options (`native-image.properties`) include full support for standard process creation flags: `--allow-incomplete-classpath`, `--report-unsupported-elements-at-runtime`, and feature flags enabling `clojure.java.shell` and runtime subprocess controls required by pods across POSIX and Windows loops.


# Project Workflow & Strict TDD Protocols

## MANDATORY: RULES OF ENGAGEMENT (NEVER VIOLATE)
1. **No Implementation Without Failing Test**: You are FORBIDDEN from writing or modifying any production code until a corresponding test has been written and HAS FAILED in the current test run.
2. **Never Generate Code & Tests Simultaneously**: You must write the test file first, execute it, observe the specific failure, and ONLY THEN implement the code.
3. **No False Positives / Mock Cheating**: Tests must validate real, expected behaviors. Empty test bodies, generic `assertTrue(true)`, or mocking the very logic under test to bypass failures is strictly prohibited.
4. **Fix All Failures Immediately**: You are never allowed to proceed or commit if there is a single failing test in the suite.

## THE 3-PHASE TDD LOOP (STRICTLY ENFORCED)

You must explicitly state which phase you are entering in your thoughts before using any tools.

### Phase 1: RED (Write Test & Confirm Failure)
1. Use `Plan Mode` to design the test cases based on the feature request.
2. Write the test cases in the appropriate test file. Do NOT touch production files.
3. Run the single test file using the test tool.
4. **Verify the Failure**: You must see the test fail with the exact error expected (e.g., `Method not found` or `Expected A but got B`). If it passes accidentally, rewrite the test.

### Phase 2: GREEN (Write Minimal Code to Pass)
1. Only after Phase 1 outputs a RED result, open the production file.
2. Write the **minimum possible code** to make that specific failing test pass. Do not over-engineer or add "future features".
3. Run the test tool again.
4. Ensure the test turns **GREEN**. If other existing tests break, fix the regression immediately.

### Phase 3: REFACTOR (Clean Code Without Breaking)
1. Review the newly added code for duplication, code smells, or architectural violations.
2. Modify the production code to improve its structure.
3. Do NOT change the test code during this phase.
4. Run the full test suite to guarantee it remains **GREEN**.

---

## ANTI-HALLUCINATION & ANTI-CHEAT CHECKSUM
* **Verification Trap**: If I suspect you are hallucinating test runs, I will ask you to output the exact terminal failure stdout snippet.
* **Incremental Commits**: After every successful Green/Refactor phase, stage and commit the changes with clear messages (`test(scope): ...` then `feat(scope): ...`).
* **Dependency Rule**: If a dependency or file is missing, the test MUST fail. Do not implement "skip if missing" safety nets.

