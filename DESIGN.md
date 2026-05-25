# 可行性分析
将 Babashka (BB) 改造并编译为动态链接库（libashka），作为 C/Zig/Rust/Swift 等宿主语言的嵌入式脚本引擎，在技术上是完全可行的，但需要深入理解 GraalVM Native Image 的 C API 机制、SCI（Small Clojure Interpreter）的运行原理以及进程间通信。

以下是对该项目的可行性与实现方案的深度技术分析：
------------------------------
## 一、 可行性深度分析## 1. 核心编译可行性 (GraalVM C API)
Babashka 是通过 GraalVM Native Image 编译为本地机器码的。

* 可行性：GraalVM 原生支持将 Clojure/Java 代码编译为动态链接库（Linux .so、macOS .dylib、Windows .dll），而不是可执行文件。
* 机制：通过在 Java/Clojure 方法上使用 @CEntryPoint 注解，GraalVM 会自动生成对应的 C 语言函数符号，暴露出 C 兼容的 ABI（应用二进制接口）。
* 跨平台与架构：GraalVM Native Image 完整支持 Linux、macOS、Windows 以及 `x86_64` 和 arm64 架构。

## 2. 宿主与脚本双向调用可行性

* 宿主调用脚本：极其简单。宿主语言通过 C ABI 调用 libashka 暴露的 C 函数，传入脚本字符串或函数名，由底层 SCI 解释执行。
* 脚本调用宿主（回调）：稍微复杂。由于 Clojure 的动态性，不能直接在 Clojure 里用传统的 with-ffi 动态回调 C。最稳妥的方案是：宿主语言在初始化时，向 libashka 注册 C 函数指针（Function Pointer）。libashka 内部将这些指针映射为 Clojure 的 Function/Macro 供脚本调用。

## 3. Babashka Pods 支持可行性

* 可行性：完全支持。
* 原理：Babashka 的 Pods 机制本质上是基于标准输入输出（stdin/stdout）和 Transit 协议的独立子进程通信。它不依赖于 bb 是一个可执行文件还是一个加载在内存中的 .so。只要 libashka 内部保留了创建子进程和管道通信（java.lang.ProcessBuilder）的能力，Pods 就能无缝工作。

------------------------------
## 二、 总体架构设计
libashka 的核心架构可以分为三层：

+-------------------------------------------------------+

|        宿主语言 (C / Zig / Rust / Swift / Go)         |
+-------------------------------------------------------+

                           |
            C ABI (函数指针 / 基础数据类型)
                           |
+-------------------------------------------------------+

|  libashka 胶水层 (GraalVM @CEntryPoint / C 结构体)     |
+-------------------------------------------------------+

                           |
                      GraalVM 上下文
                           |
+-------------------------------------------------------+

|   Babashka 核心 (SCI 解释器 / 内置 Namespaces)        |
+-------------------------------------------------------+

         |                                     |
   (进程间通信)                          (内存级调用)
         v                                     v
+-----------------+                   +-----------------+

|  External Pods  |                   |  Host Callback  |
+-----------------+                   +-----------------+

------------------------------
## 三、 核心实现方案与步骤## 步骤 1：重构 Babashka 入口为 C EntryPoint
不能再使用原有的 src/babashka/main.clj 作为 main 函数入口。需要使用 Java（或通过 Clojure Gen-class）定义一个暴露 C 接口的类。
由于 Clojure 直接编写 GraalVM CEntryPoint 语法较繁琐，建议采用 Java/Kotlin 编写外壳胶水层 + 调用 Clojure 核心 的方式：

```
package libashka;
import org.graalvm.nativeimage.IsolateThread;
import org.graalvm.nativeimage.c.function.CEntryPoint;
import org.graalvm.nativeimage.c.type.CCharPointer;
import org.graalvm.nativeimage.c.type.CTypeConversion;
import clojure.java.api.Clojure;import clojure.lang.IFn;

public class LibashkaAPI {

    // 1. 初始化隔离线程环境（GraalVM 要求）
    @CEntryPoint(name = "libashka_init")
    public static IsolateThread init() {
        // 初始化 Clojure 运行时环境
        IFn require = Clojure.var("clojure.core", "require");
        require.invoke(Clojure.read("babashka.impl.sci")); 
        return org.graalvm.nativeimage.CurrentIsolate.getCurrentThread();
    }

    // 2. 执行 Clojure 脚本字符串
    @CEntryPoint(name = "libashka_eval_string")
    public static CCharPointer evalString(IsolateThread thread, CCharPointer cScript) {
        String script = CTypeConversion.toJavaString(cScript);
        
        // 调用 Babashka 底层的 SCI 执行
        IFn evalFn = Clojure.var("babashka.impl.sci", "eval-string");
        Object result = evalFn.invoke(script);
        
        // 将结果转换为 C 字符串返回
        try (CTypeConversion.CCharPointerHolder holder = CTypeConversion.toCString(String.valueOf(result))) {
            return holder.get();
        }
    }
}
```

## 步骤 2：实现“脚本调用宿主”（宿主回调）
为了让脚本能调用 C 语言函数，需要利用 GraalVM 的 org.graalvm.nativeimage.c.function.CFunctionPointer。

1. C 接口定义：在 Java 端定义一个回调接口。

```   
   public interface HostCallback extends org.graalvm.nativeimage.c.function.CFunctionPointer {
       @org.graalvm.nativeimage.c.function.InvokeCFunctionPointer
       CCharPointer invoke(CCharPointer args);
   }
```   

2. 注册函数：暴露一个 `libashka_register_callback` 给宿主。
   
```
   @CEntryPoint(name = "libashka_register_callback")
   public static void registerCallback(IsolateThread thread, CCharPointer cName, HostCallback callback) {
       String name = CTypeConversion.toJavaString(cName);
       // 在底层 SCI 的 env 中，将该 name 映射为一个 Clojure 函数
       // 该 Clojure 函数内部通过胶水代码调用 callback.invoke(...)
   }
```   
   
## 步骤 3：数据序列化与传递
由于 C ABI 只能高效传递指针和基础类型，宿主与脚本之间复杂对象（如 Map、List）的传递建议采用以下两种方式之一：

   1. JSON / Transit (推荐)：使用 Babashka 自带的 cheshire 或 transit。宿主将结构化数据转为 JSON 字符串传给 libashka，libashka 解析为 Clojure Map；反之亦然。虽然有序列化开销，但极其稳定且跨语言通用。
   2. Raw String：简单场景直接传 String。

## 步骤 4：修改编译配置 (project.clj 或 deps.edn)
需要配置 GraalVM 的 native-image 参数，将其从构建可执行文件改为构建共享库。
在 native-image 的参数中加入：

```
--shared \
-H:Name=libashka \
-H:+API \
--no-fallback
```

* `--shared`：指定编译为动态链接库（.so/.dylib/.dll）。
* `-H:+API`：允许生成用于 C 语言的 .h 头文件。

------------------------------
## 四、 关键技术挑战与解决方案## 1. 内存管理与垃圾回收 (GC)

* 挑战：宿主语言（如 C/Zig）手动管理内存，而 libashka 内部运行着 GraalVM 的 JVM 垃圾回收器（通常是 Serial GC）。
* 生命周期：所有由 libashka 内部创建并返回给宿主的 CCharPointer（C 字符串），其内存属于 GraalVM 堆。如果直接返回，可能会被 GC 释放，导致宿主语言读到野指针。
* 解决办法：不要在 libashka 内部直接返回托管的 C 指针。应该让宿主语言负责分配内存缓冲区（Buffer），并把 Buffer 指针传给 libashka，由 libashka 将数据 memcpy 进去；或者提供一个 `libashka_free_string(void* ptr)` 函数，由宿主显式释放。

## 2. 信号冲突 (Signal Handling)

* 挑战：GraalVM 默认会接管系统的很多信号（如 SIGSEGV, SIGPIPE, SIGINT），这会和宿主语言（尤其是 Rust/Go 或复杂的 C 应用）的信号处理器发生冲突。
* 解决办法：在编译 native-image 时，必须加入参数 --install-exit-handlers 或通过设置环境变量/配置关闭信号接管（例如 Babashka 原生支持的 `BABASHKA_DISABLE_SIGNAL_HANDLERS=true`）。

## 3. 多线程与多实例 (Isolates)

* 挑战：如果宿主语言是多线程的，多个线程同时调用 libashka 会怎样？
* 解决办法：GraalVM 提供了 Isolate 机制。每个 Isolate 是一个完全隔离的独立运行时堆空间。
* 方案 A（单实例锁）：全局只创建一个 Isolate，宿主语言通过互斥锁（Mutex）保证同一时间只有一个线程在 eval 脚本。
* 方案 B（多实例，推荐）：宿主语言的每个线程在初始化时，都调用 libashka_init() 创建一个新的 IsolateThread。各线程互不干扰，完全并发，最适合作为嵌入式脚本引擎。

------------------------------
## 五、 项目推进路线图建议
若要正式启动 libashka 项目，建议分为三期推进：

1. 第一期（原型验证）：
* Fork 官方 Babashka 仓库。
   * 编写一个简单的 LibashkaAPI.java，只实现 libashka_init 和 libashka_eval_string。
   * 在 Linux/macOS 下使用 native-image --shared 成功打出 .so/.dylib。
   * 写一个极简的 main.c，使用 dlopen 加载它并成功打印出 ( + 1 2 ) 的结果 3。
2. 第二期（核心功能重构）：
* 迁移 Babashka 核心的 SCI 上下文初始化逻辑，使其能够绑定到特定的 Isolate 中。
   * 实现宿主回调注册机制（C 函数指针转 Clojure Fn）。
   * 验证 Pods 机制在动态库模式下的工作情况（确保 bb-tasks 和 process 正常）。
3. 第三期（生态与跨平台部署）：
* 配置 GitHub Actions，利用多系统 Matrix 分别在 ubuntu-latest, macos-latest, windows-latest 上编译出 x86_64 和 arm64 的二进制动态库。
* 为 Rust (libashka-sys/crate)、Zig (zig-libashka)、Swift 编写语言绑定（Bindings）和包管理支持。


# 接口定义

为了让 libashka 能够完美嵌入 C/Zig/Rust/Swift 等语言，我们需要定义一套标准的 C ABI (应用二进制接口)。这套接口将采用 C 语言语法进行规范定义，所有现代编程语言都可以无缝对接。
以下是为 libashka 量身定制的跨语言交互接口（C Header 规范）的详细定义，包含数据结构、核心 API 以及双向调用的具体实现方案。
------------------------------
## 一、 核心数据结构定义 (libashka.h)
由于跨语言调用的特殊性，直接传递复杂的指针和结构体会导致严重的内存安全问题。我们采用 Isolate 句柄机制 隔离内存，并使用 JSON/Transit 字符串 作为复杂对象传递的通用载体。

```
#ifndef LIBASHKA_H
#define LIBASHKA_H
#include <stdint.h>
#include <stddef.h>
// 1. 运行时上下文句柄（对应 GraalVM 的 IsolateThread）
typedef struct libashka_ctx_t libashka_ctx_t;
// 2. 统一的数据传输结构体（支持基础类型和 JSON 序列化）
typedef enum {
    LIBASHKA_TYPE_NIL = 0,
    LIBASHKA_TYPE_INT64 = 1,
    LIBASHKA_TYPE_FLOAT64 = 2,
    LIBASHKA_TYPE_BOOLEAN = 3,
    LIBASHKA_TYPE_STRING = 4,      // 普通字符串
    LIBASHKA_TYPE_JSON = 5         // 复杂结构（Map/List）序列化为 JSON 字符串
} libashka_type_t;
typedef struct {
    libashka_type_t type;
    union {
        int64_t i64;
        double f64;
        int32_t boolean;        // 0 为 false, 1 为 true
        const char* str;        // 字符串指针
    } value;
} libashka_val_t;
// 3. 宿主语言回调函数指针类型定义// 宿主函数接收一个上下文、参数数组和参数个数，返回一个 libashka_val_t 结果
typedef libashka_val_t (*libashka_host_fn)(libashka_ctx_t* ctx, const libashka_val_t* args, size_t arg_count);
#endif // LIBASHKA_H
```

------------------------------
## 二、 核心 API 定义（宿主控制层）
这些函数由 libashka 动态库导出，供宿主语言调用，用于生命周期管理和脚本加载。

```
// ==========================================// 1. 生命周管理// ==========================================
/**
 * 初始化一个独立的 Babashka 运行时上下文 (Isolate)
 * 每个上下文拥有独立的内存堆，支持多线程并发（一线程一上下文）
 */
libashka_ctx_t* libashka_create_context(void);

/**
 * 销毁上下文，释放其占用的所有 GraalVM 内存
 */
void libashka_destroy_context(libashka_ctx_t* ctx);

// ==========================================// 2. 内存管理（关键：由谁分配，由谁释放）// ==========================================
/**
 * 释放由 libashka 内部通过 malloc 分配并返回给宿主语言的字符串
 * 宿主语言在读取完 ASHKA_TYPE_STRING/JSON 的返回值后，必须调用此函数避免内存泄漏
 */
 void libashka_free_value(ashka_val_t val);

// ==========================================// 3. 脚本加载与执行// ==========================================
/**
 * 加载并执行一段 Clojure/Babashka 脚本字符串
 * @param ctx 运行时上下文
 * @param script 脚本代码
 * @return 执行最后一行表达式的结果
 */
libashka_val_t libashka_eval_string(libashka_ctx_t* ctx, const char* script);
/**
 * 加载并执行指定路径的脚本文件
 */
libashka_val_t libashka_eval_file(libashka_ctx_t* ctx, const char* file_path);
```

------------------------------
## 三、 进阶接口：双向函数调用## 1. 宿主语言调用脚本函数 (Host -> Script)
宿主语言需要能够调用脚本中通过 (defn ...) 定义的函数。

```
/**
 * 调用内嵌脚本中定义的函数
 * @param ctx 运行时上下文
 * @param ns_name 命名空间名称 (例如 "user" 或 "my-app.core")
 * @param fn_name 函数名称 (例如 "calculate-total")
 * @param args 参数数组
 * @param arg_count 参数数量
 * @return 脚本函数的返回值
 */
 libashka_val_t libashka_call_script_fn(
    libashka_ctx_t* ctx, 
    const char* ns_name, 
    const char* fn_name, 
    const libashka_val_t* args, 
    size_t arg_count
);
```

## 2. 脚本调用宿主函数 (Script -> Host)
通过在宿主语言中向 libashka 注册函数指针，将其映射为 Babashka/SCI 环境中的命名空间函数。

```
/**
 * 向内嵌语言环境注册一个宿主语言的函数（回调）
 * @param ctx 运行时上下文
 * @param ns_name 要注册到的脚本命名空间 (例如 "host")
 * @param fn_name 脚本中使用的函数名 (例如 "log-info")
 * @param host_fn 宿主语言的函数指针
 */
 void libashka_register_host_fn(
    libashka_ctx_t* ctx, 
    const char* ns_name, 
    const char* fn_name, 
    libashka_host_fn host_fn
);
```

------------------------------
## 四、 具体交互场景示例（以 C 语言为宿主）
为了直观展示上述接口如何协同工作，以下是一个完整的 C 语言调用逻辑伪代码：
## 场景 A：脚本调用宿主定义的加密/日志函数
1. 宿主语言（C）定义具体的函数逻辑并注册：

```
// 宿主语言编写的打印函数，供脚本调用
libashka_val_t my_host_logger(libashka_ctx_t* ctx, const libashka_val_t* args, size_t arg_count) {
    if (arg_count > 0 && args[0].type == LIBASHKA_TYPE_STRING) {
        printf("[Host Log]: %s\n", args[0].value.str);
    }
    libashka_val_t ret = { .type = LIBASHKA_TYPE_NIL };
    return ret;
}
int main() {
    libashka_ctx_t* ctx = libashka_create_context();
    
    // 注册到内嵌语言中，注册后脚本里可以用 (host/log "hello") 调用
    libashka_register_host_fn(ctx, "host", "log", my_host_logger);
    
    // 执行一段调用了宿主函数的 Clojure 脚本
    libashka_eval_string(ctx, "(host/log \"Hello from Babashka script!\")");
    
    libashka_destroy_context(ctx);
    return 0;
}
```

## 场景 B：宿主调用脚本中定义的复杂业务逻辑（传递 JSON）
2. 脚本文件 business.clj 内容：

```
(ns business
  (:require [cheshire.core :as json]))
;; 定义一个处理用户数据的函数
(defn process-user [json-str]
  (let [user (json/parse-string json-str true)
        updated-user (assoc user :status "active" :vip true)]
    (json/generate-string updated-user)))
```

3. 宿主语言（C）加载并调用该函数：

```
int main() {
    libashka_ctx_t* ctx = libashka_create_context();
    
    // 1. 加载脚本文件
    libashka_free_value(libashka_eval_file(ctx, "./business.clj"));
    
    // 2. 准备传入的 JSON 数据
    const char* input_json = "{\"name\": \"Alice\", \"age\": 30}";
    libashka_val_t arg0 = {
        .type = LIBASHKA_TYPE_JSON,
        .value.str = input_json
    };
    
    // 3. 调用脚本中的 business/process-user 函数
    libashka_val_t result = libashka_call_script_fn(ctx, "business", "process-user", &arg0, 1);
    
    // 4. 处理返回值
    if (result.type == LIBASHKA_TYPE_JSON || result.type == LIBASHKA_TYPE_STRING) {
        printf("Result from Script: %s\n", result.value.str);
    }
    
    // 5. 必须由宿主释放返回值内存（因为内部通过 memcpy 复制到了新堆区）
    libashka_free_value(result);
    
    libashka_destroy_context(ctx);
    return 0;
}
```

------------------------------
## 五、 内部核心实现提示 (GraalVM Java 侧)
在 libashka 的 Java/Clojure 胶水层中，处理 `libashka_val_t` 结构体转换时需要遵循以下原则：

   1. 基本类型直接映射：`int64_t` 映射为 Java 的 long，double 映射为 double。
   2. 字符串安全隔离：
   * 当接收宿主传入的 const char* 时，使用 CTypeConversion.toJavaString() 转换为 Java 字符串。
      * 当返回给宿主字符串时，不能直接返回指向 Java 堆的指针。应该在 Java 侧申请一个 C 堆空间的内存块 (Unsafe.allocateMemory 或 GraalVM 提供的 C 分配器)，将字符串复制过去，再封装进 libashka_val_t 返回。宿主语言调用 libashka_free_value 时，再显式释放这块 C 堆内存。
   
这套接口通过精简的 C 类型和序列化协议，彻底规避了不同语言特有的对象生命周期冲突问题。

# 目录组织结构

你的全新 libashka 仓库结构应该像下面这样干净：

```
libashka/                  # 这是一个全新的、独立的 Git 仓库！
├── deps.edn               # 独立配置文件：在这里直接依赖官方的 Babashka
├── src-java/
│   └── libashka/
│       ├── LibashkaStructs.java  # 你的 C 结构体映射
│       └── LibashkaAPI.java      # 你的 C EntryPoint 核心
├── src-clojure/
│   └── libashka/
│       └── core.clj              # 你的扩展和代理逻辑
└── test-host/
    └── main.c                    # 你的宿主测试程序
```

## 此时你的 deps.edn 应该这样写：
你不需要拷贝官方的 deps.edn，只需要把它当作 Git 依赖或者 Maven 依赖引入：

```
{:paths ["src-clojure" "src-java"]
 :deps
 {;; 核心：直接将官方 Babashka 仓库的某个特定 Tag 或 Commit 作为依赖引入！
  org.babashka/babashka {:git/url "https://github.com"
                         :git/sha "官方最新的某个Commit_SHA_或者Tag"}
  
  ;; 如果官方某些内部类没有暴露，可以直接把 SCI 乱入进来
  org.babashka/sci {:mvn/version "RELEASE"}}

 :aliases
 {:libashka
  {:exec-fn libashka.build/build-shared-lib
   :exec-args {:graalvm-args ["--shared" "-H:Name=libashka" "--no-fallback"]}}}}
```

------------------------------
## 三、 为什么“影子项目”的维护代价最小？

   1. 绝对的零冲突（Zero Conflict）：
   你没有修改 Babashka 的一行源码。官方怎么重构、怎么改动文件名，都与你的仓库无关。你和官方之间唯一的纽带是 deps.edn 里的那行 Commit SHA。
   2. 升级代价极低（只需改一行字符）：
   当 Babashka 发布了新版本，或者修复了某个安全漏洞，你只需要去官方仓库复制一下最新的 Commit SHA 或者 Tag，粘贴到你自己的 deps.edn 里。然后运行 make 重新编译。如果编译通过，升级就完成了。
   3. 极其适合 CI/CD（自动化流水线）：
   因为项目是一个完全标准的、自包含的 Clojure/Java 项目，GitHub Actions 可以在没有任何前置条件（比如去 apply 补丁、或者处理 rebase 冲突）的情况下，直接拉取代码、下载依赖并调用 GraalVM 编译。
   4. 方便向上游贡献（Upstream）：
   如果未来某一天，你发现为了实现更高效的 C 调用，必须改动 Babashka 核心的某些私有方法（private 改为 public），你可以在官方仓库提一个干净的 PR。在 PR 被合并前，你可以在自己的 deps.edn 里临时把 git/url 指向你自己临时修 bug 的 fork 分支，两不耽误。

------------------------------
## 四、 项目推进建议

* 第一步：建立独立的 libashka 仓库。
* 第二步：在 deps.edn 中使用 git/deps 引入官方 Babashka。
* 第三步：在 LibashkaAPI.java 中，尝试通过类加载器和 Clojure 交互接口（clojure.java.api.Clojure）去调用 Babashka 的核心启动类（例如 babashka.main 或 babashka.impl.sci）。


# 代码实现
为了让 libashka 项目能够顺利编译，我们需要在原版的 Babashka 项目结构中合理嵌入 Java 胶水层代码、配置文件以及用于测试的 C 代码。
以下是推荐的完整项目文件目录树及各个文件的具体路径：

```
libashka/
├── deps.edn                  # 核心配置文件：引入 Babashka 依赖与编译别名
├── src/                      # Clojure 扩展层
│   └── libashka/
│       └── impl.clj          # 构建脚本与内部代理逻辑, Clojure 侧的注册回调代理逻辑
├─ src-java/                  # Java/GraalVM 胶水层源码目录
│   └── libashka/
│       ├── LibashkaStructs.java # [新加] C 结构体映射定义
│       └── LibashkaAPI.java
├── test-host/                # 宿主测试环境
│   ├── main.c                # [新加] C EntryPoint 核心 API 实现
│   ├── libashka.h            # (编译后生成到 target/，这里仅作示意)
│   └── Makefile
└── target/               # 编译输出目录 (自动生成)
    ├── libbabashka.h        # GraalVM 自动生成的 C 头文件
    ├── libbabashka.so       # Linux 动态链接库
    ├── libbabashka.dylib    # macOS 动态链接库
    └── libbabashka.dll      # Windows 动态链接库
```

## 各核心文件具体路径与职责说明## 1. Java 胶水层 (数据结构映射)

* 路径：src-java/libashka/LibashkaStructs.java
* 职责：利用 GraalVM 的 @CStruct 注解，精确对齐并模拟 C 语言中的 libashka_val_t 结构体和联合体（Union）的内存布局。

## 2. Java 胶水层 (导出 API)

* 路径：src-java/libashka/LibashkaAPI.java
* 职责：声明 libashka_create_context、libashka_eval_string 等导出符号。负责在 C 堆区分配非托管内存（UnmanagedMemory）以及调用 Clojure 的 clojure.java.api.Clojure 运行时。

## 3. Clojure 内部代理

* 路径：src/libashka/impl/libashka.clj
* 职责：在 Babashka 内置的 SCI（Small Clojure Interpreter）环境中创建一个特殊的 Namespace（如 host），将 Java 层传下来的 C 函数指针包装为标准的 Clojure Fn，供脚本直接调用。

## 4. 构建配置修改

* 路径：deps.edn (项目根目录)
* 职责：原版的 deps.edn 主要配置了 Clojure 依赖。你需要在此文件的 :paths 中加入 "src-java" 目录，确保 GraalVM native-image 工具在编译时能够扫描到其中的 Java 类文件。

## 5. 宿主测试环境

* 路径：test-host/main.c
* 职责：独立的 C 宿主程序。它不依赖 Java 环境，而是通过 #include "../target/libashka.h" 并链接 libashka.so/dylib 来模拟第三方语言（如 Rust/Zig 的底层）嵌入加载 Babashka 脚本的完整生命周期。

以下是该架构的完整实现代码，包含 Java/Clojure 胶水层（利用 GraalVM Native Image C API 编写） 和 C 语言测试宿主程序。
------------------------------
## 一、 Java / GraalVM 胶水层实现
这是 libashka 的核心。我们使用 Java 定义 C 结构体映射，并通过 @CEntryPoint 暴露 C 函数符号。
## 1. 定义 C 结构体映射 (LibashkaStructs.java)

```
package libashka;
import org.graalvm.nativeimage.c.CContext;
import org.graalvm.nativeimage.c.struct.CField;
import org.graalvm.nativeimage.c.struct.CStruct;
import org.graalvm.nativeimage.c.type.CCharPointer;
import org.graalvm.word.PointerBase;

public class LibashkaStructs {

    // 映射 C 中的枚举常量
    public static final int TYPE_NIL = 0;
    public static final int TYPE_INT64 = 1;
    public static final int TYPE_FLOAT64 = 2;
    public static final int TYPE_BOOLEAN = 3;
    public static final int TYPE_STRING = 4;
    public static final int TYPE_JSON = 5;

    // 映射 C 中的 libashka_val_t 结构体
    @CStruct("libashka_val_t")
    public interface LibashkaVal extends PointerBase {
        @CField("type") int getType();
        @CField("type") void setType(int type);

        // 模拟 Union 的字段访问
        @CField("value.i64") long getI64();
        @CField("value.i64") void setI64(long value);

        @CField("value.f64") double getF64();
        @CField("value.f64") void setF64(double value);

        @CField("value.boolean") int getBoolean();
        @CField("value.boolean") void setBoolean(int value);

        @CField("value.str") CCharPointer getStr();
        @CField("value.str") void setStr(CCharPointer value);
    }
}
```

## 2. 定义宿主回调接口与 API 实现 (LibashkaAPI.java)

```
package libashka;
import org.graalvm.nativeimage.IsolateThread;
import org.graalvm.nativeimage.c.function.CEntryPoint;
import org.graalvm.nativeimage.c.function.CFunctionPointer;
import org.graalvm.nativeimage.c.function.InvokeCFunctionPointer;
import org.graalvm.nativeimage.c.type.CCharPointer;
import org.graalvm.nativeimage.c.type.CTypeConversion;
import org.graalvm.nativeimage.c.struct.CField;
import org.graalvm.nativeimage.c.struct.CStruct;
import org.graalvm.word.PointerBase;
import org.graalvm.nativeimage.UnmanagedMemory;
import clojure.java.api.Clojure;
import clojure.lang.IFn;
public class LibashkaAPI {

    // 定义宿主语言的回调函数指针接口
    public interface HostFnPointer extends CFunctionPointer {
        @InvokeCFunctionPointer
        void invoke(IsolateThread thread, LibashkaStructs.LibashkaVal result, LibashkaStructs.LibashkaVal args, long argCount);
    }

    // 1. 初始化并创建上下文
    @CEntryPoint(name = "libashka_create_context")
    public static IsolateThread createContext() {
        IsolateThread thread = org.graalvm.nativeimage.CurrentIsolate.createIsolate();
        // 预加载 Babashka / SCI 环境
        IFn require = Clojure.var("clojure.core", "require");
        require.invoke(Clojure.read("babashka.impl.sci")); 
        return thread;
    }

    // 2. 销毁上下文
    @CEntryPoint(name = "libashka_destroy_context")
    public static void destroyContext(IsolateThread thread) {
        org.graalvm.nativeimage.CurrentIsolate.detachCurrentThread();
    }

    // 3. 内存释放
    @CEntPoint(name = "libashka_free_value")
    public static void freeValue(LibashkaStructs.LibashkaVal val) {
        if (val.getType() == LibashkaStructs.TYPE_STRING || val.getType() == LibashkaStructs.TYPE_JSON) {
            CCharPointer ptr = val.getStr();
            if (ptr.isNonNull()) {
                UnmanagedMemory.free(ptr); // 释放非托管的 C 堆内存
            }
        }
    }

    // 4. 执行脚本字符串
    @CEntryPoint(name = "libashka_eval_string")
    public static void evalString(IsolateThread thread, LibashkaStructs.LibashkaVal outVal, CCharPointer cScript) {
        String script = CTypeConversion.toJavaString(cScript);
        
        // 调用 Babashka 底层 SCI
        IFn evalFn = Clojure.var("babashka.impl.sci", "eval-string");
        Object res = evalFn.invoke(script);
        
        // 将 Clojure 返回对象填入 C 结构体
        convertToCVal(res, outVal);
    }

    // 5. 注册宿主回调函数
    @CEntryPoint(name = "libashka_register_host_fn")
    public static void registerHostFn(IsolateThread thread, CCharPointer cNs, CCharPointer cName, HostFnPointer hostFn) {
        String ns = CTypeConversion.toJavaString(cNs);
        String name = CTypeConversion.toJavaString(cName);

        // 调用 Clojure 代码，在 SCI 的 env 中插入对应的代理函数
        IFn registerFn = Clojure.var("babashka.impl.sci", "register-host-callback");
        registerFn.invoke(ns, name, new clojure.lang.AFn() {
            @Override
            public Object invoke(Object args) {
                // 此处省略将 Clojure List 转为 C 数组并回调 hostFn 的胶水逻辑
                return null;
            }
        });
    }

    // 内部工具：将 Java/Clojure 对象转换为非托管的 C 结构体
    private static void convertToCVal(Object obj, LibashkaStructs.LibashkaVal outVal) {
        if (obj == null) {
            outVal.setType(LibashkaStructs.TYPE_NIL);
        } else if (obj instanceof Long) {
            outVal.setType(LibashkaStructs.TYPE_INT64);
            outVal.setI64((Long) obj);
        } else if (obj instanceof Double) {
            outVal.setType(LibashkaStructs.TYPE_FLOAT64);
            outVal.setF64((Double) obj);
        } else if (obj instanceof Boolean) {
            outVal.setType(LibashkaStructs.TYPE_BOOLEAN);
            outVal.setBoolean((Boolean) obj ? 1 : 0);
        } else {
            // 复杂类型默认作为字符串或者由 Cheshire 转为 JSON 字符串
            outVal.setType(LibashkaStructs.TYPE_STRING);
            String str = obj.toString();
            
            // 关键：在 C 堆区分配独立内存，拷贝字符串，防止被 JVM GC 回收
            byte[] bytes = str.getBytes();
            CCharPointer cStr = UnmanagedMemory.calloc(bytes.length + 1);
            for (int i = 0; i < bytes.length; i++) {
                cStr.write(i, bytes[i]);
            }
            cStr.write(bytes.length, (byte) 0); // 终止符
            outVal.setStr(cStr);
        }
    }
}
```

------------------------------
## 二、 C 语言宿主测试程序 (main.c)
这是宿主语言端的完整接入代码。它展示了如何加载库、管理上下文、传递重命名后的 libashka_val_t，以及手动管理返回值内存。

```
#include <stdio.h>
#include <stdint.h>
#include <stddef.h>

// 1. 完全对齐的类型定义
typedef struct libashka_ctx_t libashka_ctx_t;

typedef enum {
    LIBASHKA_TYPE_NIL = 0,
    LIBASHKA_TYPE_INT64 = 1,
    LIBASHKA_TYPE_FLOAT64 = 2,
    LIBASHKA_TYPE_BOOLEAN = 3,
    LIBASHKA_TYPE_STRING = 4,
    LIBASHKA_TYPE_JSON = 5
} libashka_type_t;

typedef struct {
    libashka_type_t type;
    union {
        int64_t i64;
        double f64;
        int32_t boolean;
        const char* str;
    } value;
} libashka_val_t;

// 2. 声明 libashka 导出的 C 函数（通常放在生成的 libashka.h 中）

libashka_ctx_t* libashka_create_context(void);
void libashka_destroy_context(libashka_ctx_t* ctx);
void libashka_free_value(libashka_val_t val);
void libashka_eval_string(libashka_ctx_t* ctx, libashka_val_t* out_val, const char* script);
// 3. 宿主实现的回调函数原型
void my_host_logger(libashka_ctx_t* ctx, libashka_val_t* out_val, const libashka_val_t* args, size_t arg_count) {
    if (arg_count > 0 && args[0].type == LIBASHKA_TYPE_STRING) {
        printf("[Host Success Log]: %s\n", args[0].value.str);
    }
    out_val->type = LIBASHKA_TYPE_NIL; // 回调没有返回值
}
int main() {
    printf("--- Initializing libashka ---\n");
    // 创建环境
    libashka_ctx_t* ctx = libashka_create_context();
    if (!ctx) {
        fprintf(stderr, "Failed to create context.\n");
        return 1;
    }

    // 分配用于接收结果的结构体（由宿主栈分配）
    libashka_val_t result;

    // 示例 1: 执行数学计算获取数字
    printf("\nExecuting script (Math):\n");
    libashka_eval_string(ctx, &result, "(+ 10 20)");
    if (result.type == LIBASHKA_TYPE_INT64) {
        printf("Result from Babashka: %lld\n", result.value.i64);
    }
    libashka_free_value(result); // 安全释放（虽无内部 C 堆内存，但保持良好的规范）

    // 示例 2: 执行字符串拼接获取复杂字符串
    printf("\nExecuting script (String Manipulation):\n");
    libashka_eval_string(ctx, &result, "(str \"Version: \" (clojure-version))");
    if (result.type == LIBASHKA_TYPE_STRING) {
        printf("Result from Babashka: %s\n", result.value.str);
    }
    // 关键步骤：释放由 libashka 内部利用 UnmanagedMemory 拷贝出的 C 字符串
    libashka_free_value(result); 

    // 示例 3: 释放环境
    printf("\n--- Destroying libashka context ---\n");
    libashka_destroy_context(ctx);
    printf("Done.\n");

    return 0;
}
```

------------------------------
## 三、 编译流水线参考指令
要在本地将上述代码打包成动态链接库，可以通过类似以下的流程：

1. 编译 Java 为 Native 动态库（以 Linux 为例）：
  
``` 
   native-image \
       --shared \
       -cp ./target/babashka-core.jar:./libashka \
       -H:Name=libashka \
       -H:Features=org.graalvm.nativeimage.c.CContext \
       --no-fallback
```   

   这会输出 libashka.so 和包含了 libashka_val_t 等定义的自动生成头文件 libashka.h。
   2. 编译 C 宿主程序并链接：

```   
   gcc main.c -L. -lashka -o host_app# 运行测试
   LD_LIBRARY_PATH=. ./host_app
```    

## 修改 deps.edn 配置文件

在原版 Babashka 的 deps.edn 基础上，我们需要做两件事：
1. 将存放 Java 胶水层代码的 "src-java" 目录加入到 :paths 中。
2. 添加一个专用的别名（Alias）如 :libashka，配置 GraalVM native-image 工具编译共享库所需的参数。

```
{:paths ["src" "src-java"]

 :deps
 {;; 1. 核心依赖：直接指向官方 Babashka 仓库
  ;;    建议锁定一个稳定的 Tag 或 Commit SHA，以便生产环境稳定复现
  org.babashka/babashka
  {:git/url "https://github.com/babashka/babashka"
   :git/tag "v1.3.190"      ;; 请根据实际情况替换为最新稳定版 tag
   :git/sha "c0c582b..."}   ;; 若用 tag 可省略 sha，但在 CI 中推荐用 sha

  ;; 2. 引入 GraalVM SDK 以支持 @CEntryPoint 注解
  org.graalvm.sdk/graal-sdk {:mvn/version "23.1.1"}}

 :aliases
 {:build
  {:exec-fn libashka.impl/build-shared-lib
   :exec-args {:lib-name "libashka"}}}}
```

## 2. src/libashka/impl.clj (构建脚本与代理)
这个文件承担两个职责：(1) 调用 native-image 进行编译；(2) 提供 Java 层回调 Clojure 的桥梁。
```
(ns libashka.impl
  (:require [babashka.main :as bb]
            [clojure.java.shell :refer [sh]]
            [clojure.string :as str]))
;; --- 1. 供 Java 层调用的内部代理函数 ---

(defn init-sci []
  ;; 初始化 Babashka 的 SCI 环境
  ;; 这里复用 babashka.main 的部分初始化逻辑，或者手动 require 核心命名空间
  (require '[babashka.impl.sci :as sci])
  ;; 返回一个初始化的环境对象 (atom)
  (sci/init {}))
;; --- 2. 构建脚本 (Build Script) ---

(defn build-shared-lib [{:keys [lib-name]}]
  (let [classpath (str/trim (:out (sh "clojure" "-Spath")))
        cmd ["native-image"
             "--shared"
             "-H:Name=target/libashka"     ;; 输出文件路径
             "-cp" classpath
             "--no-fallback"
             "-H:+API"                     ;; 生成 C 头文件
             "libashka.LibashkaAPI"]]      ;; 指定 EntryPoint 类
    (println "Building shared library with command:")
    (println (str/join " " cmd))
    (let [{:keys [exit out err]} (apply sh cmd)]
      (println out)
      (println err)
      (System/exit exit))))
```
