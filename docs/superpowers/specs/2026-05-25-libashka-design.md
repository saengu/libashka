# libashka Design Specification

**Date**: 2026-05-25
**Status**: Draft
**Project**: libashka — Babashka as an embeddable shared library

## 1. Overview

libashka compiles Babashka into a C-linkable shared dynamic library using GraalVM Native Image. Host applications (C, Zig, Rust, Swift, Go) embed a full Clojure/SCI interpreter via a clean C ABI, with per-thread Isolate-based execution contexts, bidirectional host-script callbacks, and full Pod support.

### 1.1 Output Artifacts

| Platform | Shared Library | 
|----------|---------------|
| Linux x86_64 / arm64 | `libbabashka.so` |
| macOS x86_64 / arm64 | `libbabashka.dylib` |
| Windows x86_64 | `libbabashka.dll` |

Public header: `include/libbabashka.h`

### 1.2 Key Design Decisions

- **Integration**: Git submodule of babashka (not a fork), inheriting its entire build pipeline
- **Isolate model**: Per-thread independent GraalVM Isolates, managed via `ConcurrentHashMap`
- **Data exchange**: JSON strings across the FFI boundary via `cheshire.core`
- **Callbacks**: `@CFunctionPointer` + `@InvokeCFunctionPointer`, JSON-in / JSON-out
- **Build flow**: `babashka uberjar → native-image --shared → libbabashka.so`
- **Pod support**: Full — preserves `babashka.pods` namespace and subprocess management

## 2. Directory Structure

```
libashka/
├── babashka/                       ← git submodule（不修改）
├── deps.edn                        ← :local/root "babashka"
│
├── src/java/libashka/
│   ├── LibashkaStructs.java        ← @CStruct 映射 libashka_val_t
│   └── LibashkaAPI.java            ← @CEntryPoint 导出函数（6个）
│
├── src/clojure/libashka/
│   ├── core.clj                    ← SCI 上下文生命周期管理
│   ├── serialization.clj           ← JSON 序列化/反序列化（cheshire）
│   ├── callbacks.clj               ← 宿主回调注册与调用调度
│   └── build.clj                   ← native-image 构建脚本
│
├── include/
│   └── libbabashka.h               ← 公开 C ABI 头文件
│
├── test/
│   ├── clojure/
│   │   └── libashka/
│   │       ├── core_test.clj       ← 上下文创建/销毁/求值
│   │       ├── serialization_test.clj
│   │       ├── callbacks_test.clj
│   │       └── pod_test.clj        ← Pod 功能测试
│   └── integration/
│       ├── c/
│       │   ├── main.c              ← C 集成测试（含 Pod 场景）
│       │   └── Makefile
│       ├── zig/
│       │   ├── main.zig            ← Zig 集成测试（含 Pod 场景）
│       │   └── build.zig
│       └── rust/
│           ├── main.rs             ← Rust 集成测试（含 Pod 场景）
│           └── Cargo.toml
│
├── script/
│   └── build                       ← 构建入口：uberjar → native-image --shared
│
├── Makefile                        ← 顶层快捷命令
├── CLAUDE.md                       ← 项目说明 + TDD 协议
└── DESIGN.md                       ← 原始架构设计文档
```

## 3. C ABI Specification (`libbabashka.h`)

### 3.1 Types

```c
/* Opaque context handle — one per thread/Isolate */
typedef struct libashka_ctx libashka_ctx_t;

/* Value type tags */
typedef enum {
    LIBASHKA_NIL     = 0,
    LIBASHKA_INT64   = 1,
    LIBASHKA_FLOAT64 = 2,
    LIBASHKA_BOOLEAN = 3,
    LIBASHKA_STRING  = 4,
    LIBASHKA_JSON    = 5    /* Composite types serialized as JSON */
} libashka_type_t;

/* Tagged union for return values */
typedef struct {
    libashka_type_t type;
    union {
        int64_t  i64;
        double   f64;
        int32_t  boolean;
        char    *string;    /* Caller must free via libashka_free_value */
    } value;
} libashka_val_t;

/* Host callback signature: JSON args in, JSON result out */
typedef char *(*libashka_host_fn_t)(const char *json_args);
```

### 3.2 API Functions

**Lifecycle:**
```c
libashka_ctx_t *libashka_create_context(void);
void            libashka_destroy_context(libashka_ctx_t *ctx);
```

**Evaluation:**
```c
libashka_val_t  libashka_eval_string(libashka_ctx_t *ctx, const char *code);
libashka_val_t  libashka_eval_file(libashka_ctx_t *ctx, const char *path);
```

**Bidirectional calls:**
```c
libashka_val_t  libashka_call_script_fn(libashka_ctx_t *ctx,
                                        const char *ns, const char *fn_name,
                                        const char *json_args);
void            libashka_register_host_fn(libashka_ctx_t *ctx,
                                          const char *ns, const char *fn_name,
                                          libashka_host_fn_t fn_ptr);
```

**Memory management:**
```c
void            libashka_free_value(libashka_val_t *val);
```

### 3.3 Design Rationale

- **Tagged union for primitives, JSON for composites**: Avoids complex structure mapping across FFI. int64/float64/bool/nil pass directly; vectors, maps, sets serialize to JSON strings with `LIBASHKA_JSON` tag.
- **String ownership**: All returned strings are allocated on the C heap via `UnmanagedMemory.malloc()`. The caller must call `libashka_free_value` to release.
- **Error model**: All errors caught internally, returned as `{"error": "stacktrace..."}` JSON via `LIBASHKA_JSON` type. No exceptions leak across the FFI boundary.
- **Per-thread Isolate**: Each `libashka_ctx_t` wraps a `graal_isolatet_t*` specific to its creating thread. Contexts are fully isolated — no shared mutable state.

## 4. Java Layer

### 4.1 `LibashkaStructs.java` — C Type Mappings

Uses GraalVM `@CStruct` and `@CFieldGroup` annotations to map the `libashka_val_t` C struct into Java-accessible memory layouts. This file is pure boilerplate — no business logic.

### 4.2 `LibashkaAPI.java` — C Entry Points

Six `@CEntryPoint` methods, each annotated with the corresponding C symbol name. All methods receive `IsolateThread` as the first parameter (GraalVM requirement).

| Method | C Symbol | Responsibility |
|--------|----------|---------------|
| `createContext` | `libashka_create_context` | Allocate Isolate, init SCI, return ctx handle |
| `destroyContext` | `libashka_destroy_context` | Tear down SCI, destroy Isolate, free resources |
| `evalString` | `libashka_eval_string` | C string → Java → delegate to Clojure → JSON → `LibashkaVal` |
| `evalFile` | `libashka_eval_file` | Read file, eval contents, return result |
| `registerHostFn` | `libashka_register_host_fn` | Wrap C fn pointer as `@CFunctionPointer`, store in registry |
| `callScriptFn` | `libashka_call_script_fn` | Resolve SCI ns/fn, parse JSON args, call, serialize result |
| `freeValue` | `libashka_free_value` | `UnmanagedMemory.free()` on string fields |

### 4.3 Callback Interface

```java
@CFunctionPointer
interface LibashkaHostFn extends PointerBase {
    @InvokeCFunctionPointer
    CCharPointer invoke(CCharPointer jsonArgs);
}
```

When a script calls a registered host function, SCI invokes `LibashkaHostFn.invoke()`, which GraalVM routes to the original C function pointer via `@InvokeCFunctionPointer`.

### 4.4 Context Storage

```java
private static final ConcurrentHashMap<Integer, SCIState> contexts = new ConcurrentHashMap<>();
private static final AtomicInteger ctxCounter = new AtomicInteger(0);
```

- Thread-safe by construction (`ConcurrentHashMap`)
- Integer keys returned as opaque `libashka_ctx_t*` handles
- Each entry holds: `sci.ctx`, host function registry atom, and Isolate reference

### 4.5 Error Handling

All exceptions caught at the Java layer:
```java
try {
    // ... eval logic ...
} catch (Exception e) {
    StringWriter sw = new StringWriter();
    e.printStackTrace(new PrintWriter(sw));
    return buildJsonError(sw.toString());
}
```
Returned as `libashka_val_t` with type `LIBASHKA_JSON` and value `{"error": "..."}`.

## 5. Clojure Layer

### 5.1 `libashka.core` — Context Management

Core namespace handling Isolate/SCI lifecycle:

- `create-context []` → Allocates new SCI context via `sci/init`, registers in `ctx-registry`, returns integer context ID
- `eval-string [ctx-id code]` → Looks up SCI context, calls `sci/eval-string`, wraps result in `{:result ...}` JSON. Catches all exceptions and returns `{:error ...}` JSON
- `destroy-context [ctx-id]` → Removes from registry, allowing Isolate GC

SCI initialization includes all babashka built-in namespaces, ensuring complete feature parity (including `babashka.pods`).

### 5.2 `libashka.serialization` — JSON Serialization

- `->json [v]` → `cheshire.core/generate-string`
- `<-json [s]` → `cheshire.core/parse-string` with keyword keys
- `->json-safe [v]` → Post-walks the value tree, converting non-serializable SCI types (IReified, deftype instances) to their string representations

### 5.3 `libashka.callbacks` — Host Function Registry

- `register-host-fn [ctx-id ns fn-name fn-ptr]` → Stores the `LibashkaHostFn` pointer in the context's host-fn registry atom
- `host-fn-namespaces [ctx-id]` → Returns SCI namespace definitions that inject `host.lib/invoke` as a discoverable function for scripts. When a script calls `(host.lib/invoke "my.ns" "my-fn" json-args)`, it resolves to the corresponding C function pointer

### 5.4 Data Flow

```
Host (C)                    Java (@CEntryPoint)              Clojure
------                      -------------------              --------
libashka_create_context  →  createContext()             →    core/create-context
                                ↓                              ↓
                            创建 Isolate                   sci/init + ctx-registry
                                ↓
                            返回 ctx 句柄

libashka_eval_string     →  evalString()                →    core/eval-string
                                ↓                              ↓
                            C string → Java String          sci/eval-string
                                ↓                              ↓
                            ← JSON 字符串                ←    ser/->json
                                ↓
                            构造 LibashkaVal → 返回给 C

libashka_register_host_fn → registerHostFn()            →    cb/register-host-fn
                                ↓                              ↓
                            包装 C fnPtr 为 LibashkaHostFn    存入 ctx :host-fns map
                                                               ↓
                                                          注入 SCI 命名空间 host.lib/invoke
                                                               ↓
(脚本调用 host.lib/invoke) → (SCI 触发) → .invoke(fnPtr)  →  C 宿主函数
```

## 6. Build Pipeline

### 6.1 Three Stages

**Stage 1: Uberjar** (reuses babashka build infrastructure)
```
babashka/script/uberjar
  → Leiningen with feature-flag profiles
  → reify2 bytecode generation
  → Output: babashka-standalone.jar
```

**Stage 2: Native Image** (shared library compilation)
```
native-image \
  -jar babashka-standalone.jar \
  -cp src/java:src/clojure \
  -H:Name=libbabashka \
  --shared \
  --no-fallback \
  -H:+JNI \
  -H:+ForeignAPISupport \
  -H:Path=target/ \
  --features=babashka.impl.CharsetsFeature \
  --allow-incomplete-classpath \
  --report-unsupported-elements-at-runtime \
  -H:IncludeResources=BABASHKA_VERSION \
  -H:IncludeResources=SCI_VERSION \
  [all existing babashka native-image flags...]

Output: target/libbabashka.so (Linux)
        target/libbabashka.dylib (macOS)
        target/libbabashka.dll  (Windows)
```

**Stage 3: Integration test compilation**
```
C:    gcc -ldl test/integration/c/main.c -o test/c_runner
Zig:  zig build-exe test/integration/zig/main.zig
Rust: cargo build --manifest-path test/integration/rust/Cargo.toml
```

### 6.2 `script/build`

Bash script orchestrating stages 1-2. Key environment variables:

- `BABASHKA_FEATURE_*` — Feature flags (all enabled by default, matching full babashka)
- `BABASHKA_DISABLE_SIGNAL_HANDLERS=true` — Prevents the shared library from intercepting host process signals
- `GRAALVM_HOME` — Path to GraalVM installation

### 6.3 Top-level `Makefile`

```makefile
.PHONY: build test test-clj test-c test-zig test-rust clean

build:
	./script/build

test: test-clj
	$(MAKE) -C test/integration/c test
	cd test/integration/zig && zig build test
	cd test/integration/rust && cargo test

test-clj:
	clojure -M:test

clean:
	rm -rf target/
```

## 7. Testing Strategy

### 7.1 Three-Layer Test Pyramid

| Layer | Location | Scope | % of tests |
|-------|----------|-------|------------|
| Clojure unit tests | `test/clojure/libashka/` | Context lifecycle, serialization, callbacks, Pod namespace | 55-60% |
| Integration tests | `test/integration/{c,zig,rust}/` | dlopen real .so, end-to-end scenarios | 15-20% |
| Java layer | (validated indirectly via Clojure + integration) | @CEntryPoint correctness | 20-25% |

### 7.2 Clojure Test Files

| File | What it tests |
|------|--------------|
| `core_test.clj` | `create-context`, `destroy-context`, `eval-string` basic evaluation, exception capture into error JSON, multi-context isolation |
| `serialization_test.clj` | Clojure ↔ JSON round-trip, SCI special type safety, nil/number/string/nested structure correctness |
| `callbacks_test.clj` | Host fn registration, script-calls-host flow, argument passing, return value propagation, unregistered-fn error |
| `pod_test.clj` | `babashka.pods` namespace availability, Pod loading, Pod subprocess lifecycle |

### 7.3 Integration Test Scenarios (all languages)

Each integration test suite (C, Zig, Rust) covers:

1. `test_create_destroy` — Context creation/destruction, non-NULL verification
2. `test_eval_simple` — `(+ 1 2)` → 3
3. `test_eval_error` — Syntax error → `{"error": "..."}`
4. `test_multi_contexts` — Two independent contexts, no variable cross-contamination
5. `test_host_callback` — Register C fn, call from script, verify result
6. `test_call_script_fn` — Host calls script-defined function
7. `test_pod_load` — Load external Pod, call Pod function
8. `test_pod_subprocess` — Pod spawns subprocess and communicates
9. `test_memory_leak` — Loop create/destroy, verify no memory leak
10. `test_thread_isolation` — Multi-threaded, each with own context, verify isolation

### 7.4 TDD Execution Order

**Phase 1 — Clojure layer (pure JVM, no native-image dependency):**
1. RED: Write `core_test.clj` → fail (file/fn does not exist)
2. GREEN: Implement `libashka.core` → pass
3. RED: Write `serialization_test.clj` → fail
4. GREEN: Implement `libashka.serialization` → pass
5. RED: Write `callbacks_test.clj` → fail
6. GREEN: Implement `libashka.callbacks` → pass
7. RED: Write `pod_test.clj` → fail
8. GREEN: Ensure `babashka.pods` namespace correctly injected → pass

**Phase 2 — Java @CEntryPoint layer:**
- Validated on JVM via Clojure tests that exercise the same code paths
- Full @CEntryPoint validation requires native-image compilation (Phase 3)

**Phase 3 — Integration tests (requires compiled .so):**
1. Build: `script/build` → `libbabashka.so`
2. RED: Write `main.c` → compilation against header succeeds, runtime tests fail until .so is linked
3. GREEN: Compile `main.c`, dlopen `.so`, run all scenarios → pass
4. Repeat for Zig and Rust

## 8. Pod Support

### 8.1 Requirements

- `babashka.pods` namespace must be fully available in every SCI context
- Pods communicate over stdin/stdout with their parent process
- In shared-library mode, the parent process is the host application — Pod subprocess management must be preserved
- External Pod loading via `babashka.pods/load-pod` must work

### 8.2 Implementation

- Babashka's existing Pod infrastructure is inherited unchanged
- `clojure.java.shell` and process management are enabled via native-image flags:
  - `--allow-incomplete-classpath`
  - `--report-unsupported-elements-at-runtime`
- SCI context initialization includes `babashka.pods` in the namespace map
- Pod subprocesses inherit the host process environment; no special isolation needed beyond what babashka already provides

## 9. Thread Safety & Isolation

### 9.1 Per-Thread Isolate Model

Each call to `libashka_create_context` creates a new GraalVM Isolate:
- Isolates share no mutable state
- Each Isolate has its own SCI context, heap, and host function registry
- An Isolate is bound to its creating thread (GraalVM constraint)

### 9.2 Thread Safety Guarantees

- `ConcurrentHashMap` for context registry — safe for concurrent `createContext`/`destroyContext` calls from different threads
- `AtomicInteger` for context ID generation — no duplicate IDs
- Per-context `atom` for host fn registry — lock-free per-isolate mutations
- No global mutable state outside the registry map

### 9.3 Constraints

- An Isolate cannot be shared across threads — each thread must create its own context
- GraalVM Isolate creation has a fixed memory overhead (~few MB per Isolate)
- The number of concurrent Isolates is limited by available memory

## 10. Dependencies

### 10.1 Build Dependencies

- **GraalVM** (JDK 21+ with `native-image`)
- **Leiningen** (inherited from babashka build)
- **Clojure CLI** (`deps.edn` / tools.deps)
- **GCC / Zig / Rust toolchains** (for integration tests only)

### 10.2 Runtime Dependencies

- **None** — the compiled `.so`/`.dylib`/`.dll` is a standalone native binary
- Host application links against the shared library; no JVM or Clojure runtime needed at deployment

### 10.3 deps.edn

```clojure
{:paths ["src/clojure"]
 :deps {org.babashka/babashka {:local/root "babashka"}
        cheshire/cheshire {:mvn/version "5.13.0"}
        org.clojure/clojure {:mvn/version "1.12.0"}}
 :aliases {:build {:main-opts ["-m" "libashka.build"]}
           :test  {:extra-paths ["test/clojure"]
                   :extra-deps {io.github.cognitect-labs/test-runner
                                {:git/url "https://github.com/cognitect-labs/test-runner"
                                 :git/sha "..."}}}}}
```

## 11. Open Questions & Risks

### 11.1 Risks

| Risk | Impact | Mitigation |
|------|--------|------------|
| Babashka upstream build changes break our pipeline | High | Git submodule pinned to specific SHA; CI catches breakage |
| GraalVM `--shared` conflicts with babashka native-image flags | Medium | libsci already validates `--shared` compatibility; test early |
| Pod subprocess behavior differs in shared-library context | Medium | Dedicated Pod integration tests across all 3 host languages |
| Isolate memory overhead unacceptable for high-concurrency use | Low | Document the constraint; per-thread Isolate is explicit opt-in |
| Feature flag combination explosion | Low | Default to all features enabled; document how to trim |

### 11.2 Future Considerations (out of scope for initial implementation)

- CI/CD matrix builds for all platform/arch combinations
- Pre-built binary releases (GitHub Releases)
- Language-specific binding libraries (e.g., `zig-libashka`, `rust-libashka` crates)
- Isolate pooling for reduced allocation overhead in high-frequency create/destroy scenarios
