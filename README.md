# Akash Nadackanal Vinod
## Student Number: 041156265
**Date:** February 13, 2026  

---
## Assignment 1: Serverless Computing - Critical Analysis

# Part 1: Paper Summary

Hellerstein et al. argue that first-generation serverless platforms, especially Functions as a Service (FaaS) systems such as AWS Lambda, represent both significant progress and noticeable regression at the same time [1]. Serverless simplifies deployment and scaling by removing the need to manage infrastructure. Developers only pay for what they use, and the platform handles automatic scaling. However, the authors argue that while operational simplicity improves, architectural flexibility and performance in certain scenarios decline [1].

One major limitation discussed in the paper is execution time constraints. FaaS functions are intentionally short-lived. For example, AWS Lambda historically limited execution time to fifteen minutes [1] [8]. Developers cannot assume that a function instance will remain active after execution, and state does not persist between invocations. While this design supports scalability, it becomes problematic for long-running workflows and applications that require coordination across multiple steps.

Another limitation involves communication. Functions are not directly network-addressable while running. As a result, they often communicate indirectly through storage systems or managed services [1]. This increases latency and can raise costs. The authors also describe I/O bottlenecks. Functions typically retrieve data from remote storage over the network rather than accessing local storage. As concurrency increases, bandwidth can decrease because resources are shared among multiple instances [1].
The paper introduces the idea of a data shipping problem. Instead of moving computation closer to where the data resides, serverless systems frequently require data to be transferred repeatedly into short-lived compute instances. This reduces efficiency, especially for data-heavy workloads such as analytics or machine learning [1]. This argument challenges the assumption that serverless is automatically the best choice for all cloud workloads. The authors also note limited access to specialized hardware. Early FaaS platforms mainly provided CPU and memory slices with little support for GPUs or accelerators [1]. As modern applications increasingly depend on hardware acceleration, this becomes a limitation. Finally, the paper argues that distributed systems are harder to build in FaaS environments because coordination protocols require frequent low-latency communication. When communication flows through storage services, performance and cost are negatively affected [1].

To address these challenges, the authors suggest improvements for future cloud platforms. These include better placement of code closer to data, support for heterogeneous hardware, and long-running addressable agents that behave more like traditional services while still maintaining elasticity [1]. Overall, serverless represents a meaningful step forward in operational simplicity, but it is not yet ideal for high-performance distributed computing.

---

# Part 2: Azure Durable Functions Deep Dive

Azure Durable Functions builds on standard Azure Functions by adding a workflow orchestration model. This allows developers to define structured, stateful workflows instead of treating each function as completely independent. A Durable application usually includes a client function that starts the workflow, an orchestrator function that defines the sequence of operations, and activity functions that perform the actual work [2].

## Orchestration Model

The orchestrator coordinates tasks in code. It can wait for activity results, handle retries, branch based on conditions, and run tasks in parallel [2]. In traditional serverless systems, developers often rely on queues or storage services to connect multiple function executions manually [1]. Durable Functions reduces this complexity by making coordination part of the programming model itself. This structured orchestration helps address workflow limitations present in basic FaaS systems.

## State Management

Durable Functions manages state using event sourcing and replay. The runtime records execution history and can replay the orchestrator code to rebuild its state after restarts or failures [3]. Because of this replay behavior, orchestrator code must be deterministic.

In basic FaaS platforms, developers must manually persist state between executions [1]. Durable Functions automates this process and allows workflows to behave like long-running state machines while maintaining reliability [2][3].

## Execution Timeouts

Execution time limits are a known limitation of FaaS platforms [1]. Durable Functions addresses this by dividing workflows into smaller steps. The orchestrator can pause while waiting for timers, external events, or activity results and resume later from stored state [2][4]. However, individual activity functions still operate within Azure Functions hosting plan limits [5][6]. Durable Functions reduces the impact of execution timeouts but does not eliminate them completely.

## Communication Between Functions

In basic FaaS systems, functions are not directly addressable services and often communicate through storage systems [1]. Durable Functions introduces a structured coordination model. The orchestrator schedules activity functions and receives results through the Durable runtime, which manages state and execution history automatically [2] [3]. While this improves developer experience and reliability, communication is still handled through managed infrastructure rather than direct networking.

## Parallel Execution (Fan Out and Fan In)

Durable Functions supports parallel execution using a fan out and fan in pattern. The orchestrator can start multiple activity functions simultaneously and wait for all of them to complete before combining the results [7]. This simplifies batch processing and independent task execution. However, this is task-level parallelism rather than tightly coupled distributed communication. It works well for many practical workloads but does not fully address the coordination challenges described in the paper [1].

---

# Part 3: Critical Evaluation

Azure Durable Functions clearly improves the usability of serverless platforms, especially when it comes to coordinating multi-step workflows. By introducing orchestration, deterministic replay, and automatic state management, it removes much of the manual effort developers previously needed when connecting multiple stateless functions together [2][3]. For example, instead of manually linking functions with queues and storage triggers, developers can define the workflow logic directly in an orchestrator function. This makes long-running business processes much easier to build. 
However, when examined in light of the deeper architectural concerns raised by Hellerstein et al., Durable Functions does not fundamentally eliminate several of the structural limitations identified in the paper. One limitation that remains is the disaggregated architecture model. The paper argues that serverless systems separate compute and storage in a way that forces coordination through external storage services instead of direct, low-latency communication between services [1]. Durable Functions simplifies how this coordination is written in code, but internally it still persists execution history and state to managed storage services [2][3]. For example, in an order-processing workflow where payment, inventory, and shipping steps must be coordinated, each stage still depends on stored orchestration history rather than direct service-to-service communication. The orchestrator is not a continuously running, addressable service; instead, it resumes based on stored state. This improves reliability and developer experience, but the underlying communication model still relies on storage-backed infrastructure. As a result, the I/O and latency concerns described in the paper are reduced in complexity but not removed at the architectural level. 

Another limitation that remains largely unresolved is the data locality issue, often described as the “data shipping” problem. Hellerstein et al. explain that serverless platforms frequently move data to short-lived compute instances rather than executing computation close to where the data is stored [1]. Durable Functions does not introduce mechanisms that automatically place computation near data sources. Activity functions still retrieve data from external systems such as Azure Blob Storage or databases over the network. For example, if a workflow processes large log files stored in cloud storage, each activity must download the required data before performing computation. The orchestration model defines the workflow clearly, but it does not change where the computation physically runs. For high-volume analytics or machine learning workloads, repeated data transfer can still introduce bandwidth overhead and latency, which supports the concerns raised in the paper.

In addition, Durable Functions does not significantly change hardware access limitations. While Azure Functions can scale horizontally, access to GPUs or specialized accelerators remains dependent on hosting configurations rather than being a native capability of the serverless programming model [5][6]. Orchestration can coordinate external GPU-enabled services, but this is more of an integration strategy than a redesign of the serverless infrastructure itself.

Overall, Azure Durable Functions represents meaningful progress in terms of programmability and workflow reliability. It makes stateful, long-running processes practical in a serverless environment and significantly reduces development complexity. However, it does not fundamentally change the disaggregated, storage-mediated architecture that the paper critiques. The improvements are important from a developer’s perspective, but at the infrastructure level, many of the original limitations still exist. In that sense, Durable Functions moves serverless computing forward in usability, but it does not fully overturn the structural issues described by Hellerstein et al.

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
