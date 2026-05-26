# 潜在风险与改进建议

1. Isolate 管理与 C 句柄的混淆：

系统使用一个全局的 ConcurrentHashMap<Integer, SCIState> 来管理上下文，返回给 C 的句柄 libashka_ctx_t* 实际上只是一个递增的逻辑 Integer ID（强转指针）。

设计缺陷：GraalVM Native Image 的 Isolate（隔离区）在底层要求每个调用 @CEntryPoint 的 C 函数必须显式传入底层的 IsolateThread 指针。仅传递一个 Clojure 层的自增 Integer ID 无法跨 Isolate 寻址，也无法保证 Java 层的静态 ConcurrentHashMap 能被跨隔离区正确访问。（因为每个 Isolate 拥有完全独立的堆和静态变量副本）。

重大修改建议：C 层的 libashka_ctx_t 必须直接对应真正的 GraalVM graal_isolate_t 或 graal_isolatet_t 指针。宿主每次调用必须传入该指针，从而让 GraalVM 自动切换运行时上下文。
核对 GraalVM Isolate 的上下文切换机制：不要依赖静态 ConcurrentHashMap<Integer, ...> 跨隔离区查找，而是直接将 graal_isolatet_t* 转换暴露给 C

2. Pod 进程逃逸风险

作为内嵌动态库，脚本如果拥有无限制的外部 Pod 启动权限，相当于拥有了完整的 OS 执行权限。若宿主程序将其作为“不可信脚本执行引擎”使用，将会产生重大的任意代码执行安全漏洞（RCE）。

建议：提供一个显式的权限开关或安全策略（Security Policy），允许宿主在 create_context 时控制该上下文是否禁能 Pod、是否禁能文件系统访问（即限制 SCI 的内置 Namespace）。

3. Pod 进程的生命周期孤儿问题
支持完整的 Babashka Pods，其本质是宿主应用派生的子进程。
建议：当宿主应用调用 libashka_destroy_context 销毁上下文时，必须显式且强制地终止（Kill）该上下文创建的所有 Pod 子进程 (pp. 3, 9)。否则，一旦宿主长期运行并反复创建上下文，系统中将会积攒大量无法回收的“僵尸/孤儿 Pod 进程” (p. 11)。

4. 单向的回调机制设计缺陷

单向的回调机制设计缺陷：宿主回调是通过注入 host.lib/invoke 函数实现的，脚本调用时需要显式写成 (host.lib/invoke "my.ns" "my-fn" json-args)。
建议：这破坏了 Clojure 的传统方言习惯。更合理的做法是在 SCI 初始化时，根据注册的 ns 和 fn_name，动态地在 SCI 上下文中生成对应的 Clojure 代理函数（Macro 或 Proxy Function）。这样用户在脚本中就能直接调用 (my.ns/my-fn args)，对脚本创作者更隐蔽、更友好。

6. JSON 序列化开销吞噬性能

如果宿主应用需要频繁与 Clojure 交换大规模数据（例如每秒数万次的大复杂 Map 传递），cheshire 的 JSON 序列化与反序列化将成为绝对的性能瓶颈。

建议：如果未来定位高性能场景，应在设计中为二进制序列化（如 Protocol Buffers、Transit 或 MessagePack）留出扩展接口（例如引入 LIBASHKA_BINARY 标签）。


7. Isolate 创建的冷启动与内存开销：
Isolate 线程池列为了“未来考虑” 。
审查意见：GraalVM Isolate 的创建虽然比启动 JVM 快得多，但仍有毫秒级的消耗和数兆字节的基线内存占用。如果宿主频繁地 create / destroy，性能会很差。强烈建议在初版设计中就至少提供一个基础的上下文复用机制（Context Pooling）。

	

