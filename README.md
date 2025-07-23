# Design Document — Lightweight Visual Data‑Flow IDE

## 1 Rationale and Scope

Scientific exploration often evolves through short, self‑contained code fragments that investigators wish to rearrange, recompute and visualise at will. Linear notebooks impose sequential execution and entangle code with output, hampering modular reuse. The present system offers a two‑dimensional canvas where nodes correspond to ordinary source‑level functions and edges encode data dependencies. Each node represents a pure function with immutable inputs and deterministic outputs, ensuring that data flows predictably through the computational graph, with the exception of dedicated root nodes that intentionally set global state to initialise the context. Incremental recomputation propagates only when inputs truly change, while every artefact—source, graph, intermediate result—remains plain‑text and version‑control friendly. Python and Julia are first‑class citizens from the outset; additional languages may join later.

## 2 Guiding Principles

Permanence dictates that nothing essential hides inside binary blobs. Minimal intrusion keeps users in their preferred editors; the IDE merely discovers functions. Resilience demands that long‑running numerical kernels never freeze the interface. Parity ensures symmetric capability between Python and Julia.

## 3 High‑Level Architecture

Five collaborating subsystems realise these aims.

### 3.1 Function Indexer

Whenever a workspace opens or the watcher detects a write, a Rust thread tokenises every `.py` and `.jl` file once. Python parsing relies on the standard `ast` module via PyO3; Julia parsing relies on CSTParser.jl through an embedded Julia instance initiated once per application session. Each top‑level function definition yields a manifest record containing its module path, UTF‑8 byte span, ordered parameters, default expressions as source substrings, any type annotations, and the raw docstring. A SHA‑256 digest of the byte span furnishes a stable identifier. Filtering rules, expressed as globs or regexes on module paths, apply after discovery and remain user‑configurable.

### 3.2 Runtime Scheduler and Execution Model

#### 3.2.1 Root Nodes (Context Initialisation)

Certain nodes in the graph are designated as root nodes. These serve to initialise the runtime environment by importing packages, defining constants, or performing setup operations that may mutate global state. Every other node is automatically treated as dependent on all root nodes, ensuring that any change to a root node (in code, output, or execution status) invalidates the entire dependent subgraph.

Root nodes run first, in deterministic order. Their outputs need not be used explicitly, but they form the semantic foundation for the workspace. Users are encouraged to group all global-affecting statements—such as `import numpy as np`, `plt.style.use(...)`, or `ENV[...] = ...`—into root nodes to clarify where context begins. Because these nodes are designed to produce side effects, their outputs may be ignored by the runtime, but their presence ensures stable reproducibility. The scheduler treats their mutation as an upstream event equivalent to changing a function’s return value.

Root nodes should remain lightweight and avoid generating large in-memory objects. Their purpose is to set up context, not compute results. Excessive data allocations or slow side-effects in root nodes risk degrading interactivity and should be surfaced with runtime warnings.

A single interpreter process per language lives for the entire workspace. Within Python a daemon initialises `asyncio`; within Julia the equivalent pattern emerges with `Base.@async` and Channels. The scheduler itself resides in Rust. It communicates with each daemon through a minimalist remote‑procedure protocol that follows the JSON‑RPC specification. Every call request is a tiny JSON object carrying a method name, parameters and a correlation identifier. The daemon deserialises the request, invokes the target function and serialises the result into a matching JSON reply. A local Unix‑domain socket (or named pipe on Windows) conveys the raw bytes.

Each node invocation becomes a coroutine posted to the language's event loop. Immediately before the call the runtime gathers the latest outbound objects of every parent edge and compares their identities against the node's cache. At least one change marks the node dirty and ready. Depth‑first topological traversal guarantees that a runnable node is never scheduled before its prerequisites finish. Blocking calls run on worker threads (`run_in_executor` in Python, `Threads.@spawn` in Julia) so the event loop remains responsive.

Global mutable state is monitored through bytecode instrumentation to preserve the functional paradigm while accommodating languages that permit imperative constructs. When running on CPython 3.12 or later, the same effect can be achieved more cleanly by subscribing to the `STORE` and `DELETE` event groups via the `sys.monitoring` interface (PEP 669), avoiding code object rewriting entirely. This dual-path approach ensures compatibility across Python versions while enabling low-overhead tracking of global writes in modern runtimes. Python bytecode receives patches before `STORE_GLOBAL` and `STORE_DEREF` operations; Julia intercepts through redefined `setproperty!` and `setindex!` on the `Main` module. Any detected mutation logs the affected binding and conservatively invalidates all nodes within the same module, forcing sequential re‑execution to maintain consistency. This approach captures violations that occur during function execution rather than at definition time, ensuring that dynamically constructed global references cannot escape detection. The performance penalty of instrumented execution remains negligible compared to typical numerical computation costs, while the correctness guarantee prevents subtle async‑related bugs that could corrupt intermediate results.

### 3.3 Language Back‑Ends and Data Transfer

Most scientific values are dense numeric arrays or tabular columns; Apache Arrow supplies a language‑neutral memory blueprint for these types. Whenever a Python node finishes with a NumPy array or a Julia node with a native `Array`, the daemon can wrap the underlying buffer in an Arrow header and stream only that header plus a file descriptor to the receiving process. The consumer materialises a zero‑copy view, eliminating serialisation. The entire exchange costs microseconds.

Arrow cannot faithfully represent arbitrary objects such as scikit‑learn estimators, Julia dictionaries containing functions or open file handles. During edge creation the scheduler queries the source daemon by attempting to convert the output value to Arrow format. Success enables cross‑language transfer and renders the edge in the standard visual style. Failure marks the edge as local‑only, prompting the GUI to highlight it in red and display a tooltip explaining the incompatibility. The actual Arrow conversion process determines compatibility dynamically, relying on Arrow's built‑in type system rather than maintaining an external registry of supported types. Users encountering incompatible transfers must insert explicit conversion nodes that transform complex objects into Arrow‑compatible representations before the desired cross‑language boundary.

### 3.4 Visualisation Pipeline

A node that wishes to plot returns a "visualizable" structure: text, a number, a one‑dimensional array, a two‑dimensional matrix or a three‑dimensional mesh, and similar. The GUI maps these to Matplotlib via Agg or WebAgg, to GLMakie through an OpenGL bridge or to a raw PNG texture respectively. Headless batch runs therefore reproduce identical artefacts for automated testing or continuous integration.

### 3.5 Graphical User Interface via Tauri

Tauri launches a Rust core that orchestrates sockets, file I/O and interpreter lifecycles. The diagramming surface is rendered by React Flow within the WebView context. Pan, zoom and edge routing occur entirely client side; Rust streams incremental graph and runtime state to the front end via Tauri's command channel. The WebView can schedule animation frames independently, keeping interaction smooth even when heavy kernels occupy background threads.

## 4 Persistence Format

A workspace persists as a UTF‑8 JSON document that mirrors the Obsidian Canvas schema. Each node record contains the digest, absolute file path, on‑canvas transform, frozen keyword‑argument dictionary and, when caching is enabled, a digest of its most recent value. The cached result size in bytes accompanies each node record to enable memory usage visualization within the interface. Users observe the memory footprint of individual computations through visual indicators that scale with the cached data size, providing immediate feedback about resource consumption without requiring external monitoring tools. Manual cache management operates at both individual node and workspace levels, allowing users to selectively preserve expensive intermediate results while clearing less critical cached values. This explicit approach to memory control avoids the complexity and unpredictability of automatic eviction algorithms, instead placing the optimization decisions in the hands of domain experts who understand the computational cost and reuse patterns of their specific workflows.

Each edge record lists source and target digests plus pin indices. The root object stores a graph‑wide UUID, a monotonically increasing revision counter and an optional execution log of timestamped node completions. Because the schema is declarative and stable, version control diffs remain intelligible.

## 5 Performance Profile

Direct in‑process function calls operate within the nanosecond range typical of modern interpreters. JSON‑RPC communication over Unix domain sockets introduces additional latency that will be characterized during implementation. When graphs contain hundreds of micro‑nodes, cumulative protocol overhead may approach perceptible delays; for typical scientific workloads dominated by numerical computation, communication costs remain insignificant compared to kernel execution time. MessagePack serialization could reduce encoding overhead relative to JSON while preserving the RPC protocol structure.

Arrow header exchange achieves near‑zero overhead for large array transfers through memory mapping and file descriptor passing (on POSIX) or HANDLE duplication (on Windows), ensuring that the underlying data buffers remain zero-copy. This design sidesteps the need for shared memory allocations and avoids costly serialization, while respecting operating system boundaries for secure, efficient interprocess transfer. for large array transfers through memory mapping and file descriptor passing. Type incompatibilities surface immediately during edge creation, preventing runtime surprises and guiding users toward appropriate data transformations.

## 6 Error Handling and Diagnostics

Syntax errors caught by the indexer appear as inline annotations. Runtime exceptions propagate to the scheduler which attaches the stack trace to the node and marks it as failed, while independent graph branches continue execution unimpeded. Type mismatches manifest as ordinary exceptions in the callee and are reported identically. The GUI can expose per‑node metrics such as wall time and peak memory, collected during coroutine execution, to aid optimisation.

## 7 Extension Surface

Developers may extend the front‑end visual widgets by registering new renderers that consume one of the canonical data shapes. Users extend computation simply by adding new source files; the indexer assimilates them automatically.
