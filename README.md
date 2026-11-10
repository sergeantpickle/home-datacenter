# home-datacenter

Turn a group of home computers into one logical AI compute system by composing proven distributed-inference and cluster projects instead of rebuilding them.

The project begins with AI inference because that is where the useful dependencies already exist, but the long-term boundary is broader: a home machine should be able to contribute GPU, memory, CPU, storage, and network capacity to a shared compute fabric.

## Core principle

**Do not invent infrastructure that a maintained dependency already solves. Innovate in how the pieces are composed for heterogeneous home hardware.**

Before implementing a scheduler, transport, tensor partitioner, model server, KV cache, service-discovery layer, or cluster runtime:

1. Find the maintained upstream projects that already solve the problem.
2. Test them on the actual target hardware.
3. Use them directly when possible.
4. Wrap them behind a thin adapter when their interfaces differ.
5. Fork or replace a component only when a measured requirement cannot be met upstream.

The project's value should come from integration, topology planning, lifecycle management, and a simple home-cluster experience — not from duplicating inference engines or datacenter infrastructure.

## Problem

Projects already solve important pieces of home and datacenter inference, but the pieces are disconnected.

- NVIDIA PAIR discovers and pairs local machines, manages supported engines, exposes compatible endpoints, and routes independent requests between nodes. It currently routes each request to one node; it does not pool GPU memory or shard one model across machines.
- llama.cpp RPC can expose remote devices and distribute model weights and KV cache across local and remote devices. It can use explicit tensor splits, local tensor caches, TCP, and RDMA where available.
- vLLM provides tensor, pipeline, data, and expert parallelism, including multi-node serving.
- SGLang provides tensor, data, and expert parallelism and already has supported DGX Spark workflows, including multi-Spark tensor-parallel recipes.
- NVIDIA Dynamo sits above inference engines and provides distributed-serving primitives such as routing, disaggregated prefill/decode, KV management, and planning.
- llm-d provides modular routing, KV-cache management, planning, benchmarking, and distributed-serving components.
- DeepSeek DeepInfer exposes production-scale DeepSeek inference techniques and optimized distributed execution, but its current open release has narrow hardware/model support.

The missing product is a lightweight layer that understands a house full of unlike computers and chooses the best existing engine/topology for the requested workload.

## Project idea

`home-datacenter` is a **dependency-first control plane for heterogeneous home compute**.

A node can be a DGX Spark, gaming PC, workstation, mini PC, or other supported machine. The control plane inventories what each machine can contribute and then launches an existing backend that matches the job.

The system should treat a distributed engine as a first-class resource. A PAIR-routed single-node model, a llama.cpp RPC group, a two-Spark SGLang topology, or a multi-node vLLM deployment should all look like schedulable execution targets behind one interface.

### The innovation layer

Keep the new code concentrated in five places:

1. **Capability discovery** — describe each node's hardware, memory, engines, model formats, storage, and usable network links.
2. **Topology benchmarking** — measure actual bandwidth, latency, load time, generation speed, and memory behavior instead of assuming datacenter interconnects.
3. **Backend selection** — choose an existing runtime based on model, hardware, topology, and workload.
4. **Placement planning** — decide which nodes participate and which supported parallelism strategy the selected backend should use.
5. **Lifecycle orchestration** — start, stop, monitor, recover, and expose the selected distributed service through one stable endpoint.

Do not implement tensor parallelism, expert routing, collective communication, KV-cache algorithms, inference kernels, or model formats unless an upstream dependency demonstrably cannot provide the required capability.

## Candidate dependency stack

These are candidates to validate, not mandatory permanent dependencies.

| Dependency | Use it for | Initial role |
| --- | --- | --- |
| [NVIDIA PAIR](https://github.com/NVIDIA/Personal-AI-Router) | LAN discovery, pairing, node management, request routing, compatible local endpoints | Reuse or integrate as the local control/routing layer |
| [llama.cpp](https://github.com/ggml-org/llama.cpp) RPC | Heterogeneous remote devices, distributed weights/KV, local tensor cache, optional RDMA | First pooled-memory backend |
| [vLLM](https://github.com/vllm-project/vllm) | TP/PP/DP/EP and multi-node serving | High-performance backend where hardware/model support fits |
| [SGLang](https://github.com/sgl-project/sglang) | TP/DP/EP, high-performance serving, DGX Spark support | Strong Spark and MoE backend |
| [NVIDIA Dynamo](https://github.com/ai-dynamo/dynamo) | Engine-agnostic orchestration, routing, KV management, disaggregated serving, planning | Reuse components or patterns instead of recreating datacenter primitives |
| [llm-d](https://github.com/llm-d/llm-d) | Router, distributed KV, planner, benchmark and inference analysis components | Reuse modular pieces where they remain lightweight enough |
| [DeepInfer](https://github.com/deepseek-ai/DeepInfer) | DeepSeek production inference implementation and distributed techniques | Experimental backend/reference as hardware support expands |
| [Ray](https://github.com/ray-project/ray) | Existing multi-node runtime used by vLLM and other projects | Use through upstream integrations rather than building a cluster runtime |

## First target

Prove that one model that cannot fit on a single participating machine can be served across multiple home machines **without writing a new distributed inference engine**.

The first path to test should be llama.cpp RPC because it already exposes the exact primitive:

```text
client / OpenAI-compatible consumer
              |
              v
      home-datacenter control
              |
      backend + topology plan
              |
              v
     llama.cpp coordinator
       /        |        \
      /         |         \
 DGX Spark    PC GPU     PC GPU
  memory       VRAM       VRAM
```

After that works, compare it against supported vLLM and SGLang multi-node configurations. PAIR can remain useful for node discovery, normal request routing, and eventually presenting distributed groups as virtual nodes.

## Planning model

The planner should operate on measured capabilities rather than assumptions.

Example input:

```text
model: <requested model>
workload: interactive chat
nodes:
  spark:
    memory: ...
    engine support: ...
    network: ...
  gaming-pc:
    vram: ...
    ram: ...
    engine support: ...
    network: ...
```

Example output:

```text
selected backend: llama.cpp RPC
participating nodes: spark, gaming-pc
placement: backend-supported automatic split
reason:
  - model does not fit one node
  - both nodes support selected backend
  - measured link is sufficient for this topology
fallback:
  - single-node quantized model
```

The planner should produce backend configuration, not perform model mathematics that the backend already owns.

## Heterogeneous-home constraints

Datacenter software often assumes identical accelerators, Linux everywhere, fast fabric, persistent workers, and machines dedicated to inference. A home datacenter does not.

The project should deliberately account for:

- mixed GPU generations and memory sizes;
- DGX Spark unified memory alongside discrete GPU VRAM;
- Windows and Linux nodes;
- Ethernet, Wi-Fi, Thunderbolt, ConnectX/RDMA, and other uneven links;
- computers that become unavailable because somebody starts using them;
- different model formats and backend support on different nodes;
- power, thermals, noise, and idle-only participation;
- local NVMe caches that can avoid repeatedly moving large model tensors.

This adaptation layer is where the project can contribute something new while still depending on mature execution engines underneath.

## Initial milestones

### 0. Dependency validation

Before substantial implementation:

- validate PAIR's reusable interfaces and determine whether integration should be upstream, adapter-based, or both;
- benchmark llama.cpp RPC across at least two unlike machines;
- test automatic and explicit memory splits;
- test TCP and available high-speed transports;
- validate SGLang and vLLM on the hardware combinations they currently support;
- inspect Dynamo and llm-d for components that can be reused independently without making Kubernetes mandatory;
- record what each dependency solves and what gap actually remains.

### 1. Minimal control plane

Only after validation, build the smallest missing layer:

- inventory nodes;
- inventory installed backends;
- benchmark links;
- describe candidate distributed groups;
- launch an existing backend with generated configuration;
- expose its endpoint;
- report health and resource usage.

### 2. Automatic planner

Given a model and workload, compare supported execution plans and choose one using measured data.

Possible plans may include:

- single-node execution;
- PAIR request routing;
- llama.cpp RPC pooling;
- vLLM TP/PP/DP/EP;
- SGLang TP/DP/EP;
- another proven backend discovered during research.

### 3. Broader home compute

Once inference is working, evaluate whether the same node/resource model can schedule other workloads such as ComfyUI, embeddings, speech, model conversion, LoRA training, compilation, and agent jobs.

Do not generalize before a real second workload proves the abstraction is useful.

## Non-goals

At project start, do **not**:

- create a custom distributed tensor runtime;
- create a new collective-communication library;
- implement a new inference engine;
- create another model format;
- rebuild PAIR's discovery/pairing without evidence it must be replaced;
- require Kubernetes for a small home cluster;
- create a giant permanent architecture before dependency testing shows what code is actually needed.

## Success condition

A user should eventually be able to describe the workload rather than hand-design the cluster:

```text
home-dc run <model>
```

The system discovers available resources, selects a proven backend, chooses a supported topology, launches it, and presents one endpoint.

The implementation should remain much smaller than the projects it composes.

## Current upstream references

- NVIDIA PAIR: https://github.com/NVIDIA/Personal-AI-Router
- llama.cpp RPC: https://github.com/ggml-org/llama.cpp/tree/master/tools/rpc
- vLLM distributed serving: https://docs.vllm.ai/en/latest/serving/parallelism_scaling/
- SGLang: https://github.com/sgl-project/sglang
- NVIDIA Dynamo: https://github.com/ai-dynamo/dynamo
- llm-d: https://github.com/llm-d/llm-d
- DeepInfer: https://github.com/deepseek-ai/DeepInfer
- Repository organization reference: https://github.com/sergeantpickle/repo-template
