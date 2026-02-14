# Akash Nadackanal Vinod
## Student Number: 041156265
**Date:** February 13, 2026  

---

## Part 1: Summary 

Hellerstein et al. argue that first-generation “serverless” platforms—specifically Functions-as-a-Service (FaaS) such as AWS Lambda—deliver a meaningful advance in *operational simplicity* and *autoscaling*, but regress in capabilities needed for modern computing, especially data-centric and distributed systems [1]. Their central thesis (“one step forward, two steps back”) is that autoscaling, pay-per-use execution is a real step forward, yet current FaaS designs simultaneously make it harder to build efficient data processing systems and distributed applications [1].

The paper identifies several concrete limitations typical of early FaaS offerings. First are **execution time constraints**: function instances are intentionally short-lived (e.g., 15-minute maximum in AWS Lambda at the time), and the platform cannot guarantee that follow-on requests will reuse the same warm instance—so programmers must assume state will not survive across invocations [1]. Second are **communication/network limitations**. Because functions are not directly network-addressable while running, inter-function coordination typically requires an intermediary managed service, often a storage system, which increases both latency and cost [1]. Related to this is the **I/O bottleneck**: functions access data over network links to shared services, and measured per-function bandwidth can be much lower than local storage performance. Moreover, bandwidth may degrade as concurrency scales because multiple functions share underlying host resources [1].

These design choices drive what the authors call a **“data shipping” anti-pattern**: instead of moving computation to where the data resides (as databases often do with predicate pushdown, locality-aware execution, and caching), FaaS frequently forces developers to pull data repeatedly from remote storage into ephemeral compute instances. This increases latency, reduces throughput, and can raise cost—especially for large-scale analytics or ML workloads [1]. The paper also highlights **limited hardware access**: early FaaS primarily offered CPU + memory slices with little or no access to specialized hardware (e.g., GPUs), which matters because hardware specialization is increasingly important for ML and high-performance data systems [1]. Finally, the paper argues FaaS **stymies distributed and stateful workloads** because distributed protocols (membership, leader election, coordination) require fine-grained communication; routing everything through storage intermediaries makes that communication orders of magnitude slower and often prohibitively expensive [1].

To move forward, the authors propose characteristics for future cloud programming models that preserve autoscaling while enabling high-performance, data-rich, distributed computing. Examples include: **fluid code and data placement** (supporting locality and “shipping code to data” when beneficial), **heterogeneous hardware support** (enabling accelerators and affinity-aware scheduling), and **long-running, addressable virtual agents** (durable, nameable endpoints with performance comparable to standard networking, while remaining elastically movable) [1]. They also discuss the need for more “disorderly” asynchronous programming models and better SLO-driven APIs that expose performance goals rather than only resource sizing [1].

---

## Azure Durable Functions

### 1) Orchestration model
Azure Durable Functions extends basic Azure Functions (event-triggered, short-lived compute) with a workflow/orchestration layer. A Durable app typically includes **client functions** (start/query orchestration instances), **orchestrator functions** (define the workflow in code), and **activity functions** (perform the actual work steps). The orchestrator schedules activities, waits for results, and can branch, loop, and handle retries—so the program is expressed as a coordinated workflow rather than disconnected, stateless functions [2]. This directly targets one of the paper’s pain points: without orchestration, FaaS applications often rely on external “glue” services (queues/storage) to chain steps, increasing complexity and latency [1]. Durable Functions provides a first-class composition model in code, shifting some workflow concerns from ad-hoc service stitching to a dedicated runtime [2].

### 2) State management (event sourcing, checkpointing, replay)
Durable Functions achieves “stateful workflows” by persisting execution history and reconstructing orchestrator state via **event sourcing** and **replay**. Orchestrator code is re-run (replayed) after restarts, using the stored history to rebuild local variables and progress, which is why orchestrator logic must be deterministic [3]. This responds to the paper’s criticism that first-generation FaaS forces developers to treat compute as stateless and repeatedly externalize state to slow storage [1]. Durable Functions still persists state externally (via the Durable backend), but it makes that persistence automatic and structured, allowing developers to write workflows that behave like long-running state machines [2][3]. In short: the platform keeps state durable and recoverable while letting workflow code “feel” stateful.

### 3) Execution timeouts
A major practical problem in basic FaaS is timeout-limited execution, which makes long-running processes awkward and pushes developers into brittle chaining patterns [1]. Durable Functions addresses this by letting the **orchestration** represent a long-running workflow even if no single compute activation runs for the entire duration. The orchestrator can “pause” while awaiting timers or external events and later resume from persisted history [2][4]. However, individual function executions still run under the Azure Functions hosting plan’s execution rules (i.e., timeouts and host behaviors vary by plan), and Durable Functions treats timeouts similarly to exceptions [5][6]. Conceptually, Durable Functions reduces the need for one uninterrupted long execution by splitting work into resumable steps, but it does not magically remove all per-execution limits—especially for long-running activity steps that must still complete within the platform constraints [5][6].

### 4) Communication between functions
The paper criticizes FaaS because functions are not directly addressable and often must communicate via slow storage intermediaries [1]. Durable Functions does not make activity functions directly addressable like long-lived network services; instead it provides a structured internal communication model: orchestrators schedule activities and receive results through the Durable runtime, with all coordination recorded in the orchestration history [2]. This replaces “DIY” communication patterns (e.g., hand-managed queue messages and blob writes) with a runtime-managed protocol. Practically, developers still rely on managed infrastructure under the hood, but the durable task framework abstracts it and reduces accidental complexity in passing state/results between steps [2][3]. So, Durable Functions improves *developer ergonomics and correctness* for multi-step workflows, but it doesn’t fully deliver the paper’s ideal of low-latency, direct function-to-function networking [1].

### 5) Parallel execution (fan-out/fan-in)
A key concern in the paper is that FaaS struggles with efficient distributed computing when coordination requires slow storage-based communication [1]. Durable Functions supports a **fan-out/fan-in** pattern where the orchestrator starts many activity functions concurrently and then awaits their completion to aggregate results [7]. This is closer to a “distributed task” model than naïve event chaining because the orchestration runtime tracks pending tasks and their outcomes as part of durable state, rather than forcing the developer to implement their own coordination logic with queues/blobs [2][7]. While it still isn’t the same as tightly-coupled distributed systems with direct messaging, it does enable scalable parallelism for many practical workloads (batch processing, data transforms, multi-item processing) with less coordination overhead in application code [2][7].

---

## Critical Evaluation

Azure Durable Functions represents meaningful progress on one major class of serverless limitation: **composability of stateful, long-running workflows**. In the Hellerstein et al. framing, first-generation FaaS often forces state management and coordination into external services, producing high latency, high cost, and complex “glue code” [1]. Durable Functions addresses that pain by providing a workflow runtime that persists state automatically (event sourcing + replay), offers first-class patterns (timers, external events, fan-out/fan-in), and makes multi-step orchestration a core programming model rather than an improvised architecture [2][3][4][7]. For many enterprise workloads—approvals, multi-step ETL, order processing, human-in-the-loop flows—this is a practical improvement that reduces brittleness and improves reliability.

However, at least two of the paper’s deeper criticisms remain unresolved or only partially addressed.

**(1) Direct addressability and low-latency communication remain limited.** The paper argues that non-addressable functions force communication “through slow storage,” blocking efficient distributed protocols and fine-grained coordination [1]. Durable Functions improves how developers *express* coordination, but the underlying communication model is still mediated by the orchestration backend and durable history rather than direct, low-latency networking between long-lived agents. Orchestrators must be deterministic and are replayed, which is powerful for reliability but also signals that the system is not operating as a mesh of directly communicating services [3]. In other words, Durable Functions streamlines storage-mediated coordination rather than replacing it with the paper’s vision of **long-running, addressable virtual agents** with network-like performance [1].

**(2) The “data shipping” problem is largely outside Durable Functions’ scope.** The paper’s biggest architectural critique is that FaaS tends to move data to ephemeral compute instead of moving computation toward data, which hurts performance for data-intensive workloads [1]. Durable Functions does not fundamentally change data locality: activity functions still commonly fetch from storage or databases over the network, and the orchestration layer mainly manages control flow and state transitions. Durable can reduce redundant work (e.g., caching within a warm activity instance is possible but not guaranteed), yet it does not provide automatic “fluid code and data placement” or declarative optimizations that push computation into the data plane, which the paper argues is essential for the cloud’s full potential [1]. In that sense, Durable Functions is an orchestration solution, not a data-centric execution engine.

**Verdict:** Durable Functions is best understood as “progress in workflow programming,” not a full answer to the paper’s broader agenda. It aligns with the authors’ desire for richer cloud programming models beyond bare FaaS composition—especially by making state and orchestration first-class [1][2]. But it also largely works *around* the fundamental constraints the paper highlights (addressability, efficient data locality, and the performance profile of distributed coordination). Durable Functions makes serverless more practical for stateful business processes and coarse-grained parallelism, yet it does not fully realize the paper’s envisioned future of cloud programming where code/data placement and low-latency coordination are foundational features rather than externalized compromises [1]. So: it is a genuine step forward for a specific class of workloads, but not a complete solution to “two steps back.”

---

## References

[1] Hellerstein, J. M., Faleiro, J., Gonzalez, J. E., Schleier-Smith, J., Sreekanti, V., Tumanov, A., & Wu, C. (2019). *Serverless Computing: One Step Forward, Two Steps Back (CIDR 2019).* https://www.cidrdb.org/cidr2019/papers/p119-hellerstein-cidr19.pdf  

[2] Microsoft Learn. *Durable orchestrations (Azure Functions).* https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-orchestrations  

[3] Microsoft Learn. *Durable orchestrator code constraints (determinism, replay, event sourcing).* https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-code-constraints  

[4] Microsoft Learn. *Timers in Durable Functions (durable timers).* https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-timers  

[5] Microsoft Learn. *Performance and scale in Durable Functions (function timeouts behavior).* https://docs.azure.cn/en-us/azure-functions/durable/durable-functions-perf-and-scale  

[6] Microsoft Learn. *Azure Functions scale and hosting (plan behaviors and execution considerations).* https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale  

[7] Microsoft Learn. *Fan-out/fan-in scenarios in Durable Functions.* https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-cloud-backup  

[8] AWS Documentation. *Configure Lambda function timeout (max 900 seconds / 15 minutes).* https://docs.aws.amazon.com/lambda/latest/dg/configuration-timeout.html  

[9] AWS News Blog. *AWS Lambda now supports up to 10 GB ephemeral storage.* https://aws.amazon.com/blogs/aws/aws-lambda-now-supports-up-to-10-gb-ephemeral-storage/  

---

## AI Disclosure Statement

Used ChatGPT to draft and organize the summary/analysis and to help identify relevant official documentation sources for Azure Durable Functions. I reviewed and edited the text for accuracy and ensured all claims are supported by citations.
