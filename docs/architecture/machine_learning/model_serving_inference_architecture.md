# Model Serving & Inference Optimization Architecture

## Overview

**Model Serving & Inference Optimization Architecture** defines the structural patterns for deploying AI/ML models at scale with production-grade reliability, low latency, and cost efficiency. Unlike training — which is batch-oriented and tolerates high latency — inference is real-time, user-facing, and must handle variable load with consistent performance. As LLMs dominate production AI, inference optimization has become a specialized discipline with techniques like continuous batching, KV-cache management, quantization, and speculative decoding.

This architecture covers the full inference stack: from model packaging and serving infrastructure, through request scheduling and GPU orchestration, to client-facing API design and autoscaling strategies.

Key principles:

- **Serving Is Not Training** — Inference has fundamentally different requirements: low latency, high throughput, consistent SLAs, and cost-per-token optimization
- **Hardware-Aware Design** — Inference performance is dominated by memory bandwidth and compute utilization; architecture must match workload to hardware capabilities
- **Batching Is King** — Amortizing fixed costs across multiple requests is the single largest optimization lever
- **Progressive Optimization** — Start with simple serving, then quantize, then optimize batching, then add speculative decoding — in order of complexity and impact
- **Observability First** — Measure time-to-first-token (TTFT), inter-token latency (ITL), throughput, and GPU utilization to guide optimization

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│               Model Serving & Inference Architecture                         │
│                                                                             │
│   Client Requests                                                           │
│       │                                                                     │
│       ▼                                                                     │
│   ┌──────────────┐                                                          │
│   │  API Gateway   │  (auth, rate limiting, request routing)                │
│   └──────┬───────┘                                                          │
│          │                                                                  │
│          ▼                                                                  │
│   ┌──────────────┐                                                          │
│   │  Load         │   Route to model replicas based on capacity,            │
│   │  Balancer     │   model version, and request priority                   │
│   └──────┬───────┘                                                          │
│          │                                                                  │
│          ▼                                                                  │
│   ┌──────────────────────────────────────────────────────────────────┐      │
│   │                    Inference Server                               │      │
│   │                                                                  │      │
│   │   ┌──────────────┐    ┌──────────────┐    ┌────────────────┐   │      │
│   │   │  Request       │    │  Batch        │    │  Model          │   │      │
│   │   │  Queue         │───▶│  Scheduler    │───▶│  Executor       │   │      │
│   │   └──────────────┘    └──────────────┘    └────────┬───────┘   │      │
│   │                                                     │           │      │
│   │                                            ┌────────▼───────┐  │      │
│   │                                            │  KV-Cache       │  │      │
│   │                                            │  Manager        │  │      │
│   │                                            └────────────────┘  │      │
│   └──────────────────────────────────────────────────────────────────┘      │
│          │                                                                  │
│          ▼                                                                  │
│   ┌──────────────┐    ┌──────────────────────────────────────────┐         │
│   │  Response      │    │  GPU Pool                                │         │
│   │  Streamer      │    │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐  │         │
│   └──────────────┘    │  │ GPU 0 │ │ GPU 1 │ │ GPU 2 │ │ GPU N │  │         │
│                        │  └──────┘ └──────┘ └──────┘ └──────┘  │         │
│                        └──────────────────────────────────────────┘         │
│                                                                             │
│   Cross-Cutting: ┌────────────┐  ┌──────────────┐  ┌────────────────────┐  │
│                   │ Model       │  │  Autoscaler   │  │  Metrics &         │  │
│                   │ Registry    │  │               │  │  Observability     │  │
│                   └────────────┘  └──────────────┘  └────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Core Concepts

### Inference Server Components

```
// Model Server — Core inference engine
CLASS InferenceServer
    PROPERTIES
        model          : LoadedModel
        scheduler      : BatchScheduler
        kvCacheManager : KVCacheManager
        tokenizer      : Tokenizer
        config         : ServingConfig

    FUNCTION serve(request : InferenceRequest) -> InferenceResponse
        // 1. Tokenize input
        inputTokens = tokenizer.encode(request.prompt)

        // 2. Submit to scheduler (batched with other requests)
        handle = scheduler.submit(ScheduledRequest(
            requestId = request.id,
            tokens = inputTokens,
            maxNewTokens = request.maxTokens,
            samplingParams = request.samplingParams,
            priority = request.priority
        ))

        // 3. Wait for completion or stream tokens
        IF request.stream THEN
            RETURN streamResponse(handle)
        ELSE
            RETURN awaitCompletion(handle)
        END IF
END CLASS
```

### Continuous Batching

Unlike static batching (wait for N requests, process together), continuous batching dynamically adds and removes requests from the running batch as they complete:

```
┌────────────────────────────────────────────────────────────┐
│                    Continuous Batching                       │
│                                                            │
│   Step 1:  [Req A ████████]  [Req B ██████████████]       │
│                                                            │
│   Step 2:  [Req A done]      [Req B ████████████]         │
│            [Req C ████]       ← C fills A's slot          │
│                                                            │
│   Step 3:  [Req C ████████]  [Req B done]                 │
│                              [Req D ██]   ← D fills B     │
│                                                            │
│   vs. Static:                                              │
│   Batch 1: [Req A ████████] [Req B ██████████████]        │
│            Wait for both... then                           │
│   Batch 2: [Req C ████████████] [Req D ██████]            │
│                                                            │
│   Continuous batching: ~2x higher GPU utilization          │
└────────────────────────────────────────────────────────────┘
```

```
CLASS ContinuousBatchScheduler
    PROPERTIES
        maxBatchSize    : Int
        waitingQueue    : PriorityQueue<ScheduledRequest>
        runningBatch    : List<ActiveRequest>
        kvCacheManager  : KVCacheManager

    FUNCTION step() -> List<GeneratedToken>
        // 1. Fill empty slots with waiting requests
        WHILE runningBatch.size() < maxBatchSize AND waitingQueue.isNotEmpty()
            request = waitingQueue.dequeue()
            IF kvCacheManager.canAllocate(request.estimatedBlocks) THEN
                blocks = kvCacheManager.allocate(request.estimatedBlocks)
                runningBatch.ADD(ActiveRequest(request, blocks))
            ELSE
                // Preempt lowest-priority running request if needed
                IF request.priority > runningBatch.lowestPriority() THEN
                    preempted = runningBatch.removeLowPriority()
                    kvCacheManager.free(preempted.blocks)
                    waitingQueue.enqueue(preempted.request)
                    blocks = kvCacheManager.allocate(request.estimatedBlocks)
                    runningBatch.ADD(ActiveRequest(request, blocks))
                ELSE
                    waitingQueue.requeue(request)
                    BREAK
                END IF
            END IF
        END WHILE

        // 2. Run one forward pass for the entire batch
        outputs = model.forward(runningBatch)

        // 3. Process outputs — remove completed requests
        results = EMPTY LIST
        FOR EACH active, output IN ZIP(runningBatch, outputs)
            active.generatedTokens.ADD(output.token)
            IF output.isEOS OR active.reachedMaxTokens() THEN
                results.ADD(active.finalize())
                kvCacheManager.free(active.blocks)
                runningBatch.REMOVE(active)
            END IF
        END FOR

        RETURN results
END CLASS
```

### KV-Cache Management

During autoregressive generation, key-value pairs from previous tokens are cached to avoid recomputation. Managing this cache is critical for throughput:

```
// PagedAttention — Allocate KV cache in fixed-size blocks (pages)
CLASS PagedKVCacheManager
    PROPERTIES
        blockSize    : Int = 16          // Tokens per block
        totalBlocks  : Int               // Total GPU memory / block size
        freeBlocks   : Set<Int>          // Available block indices
        blockTable   : Map<RequestId, List<Int>>   // Request → block mapping

    FUNCTION allocate(numBlocks : Int) -> List<Int>
        IF freeBlocks.size() < numBlocks THEN
            THROW InsufficientMemory
        END IF
        allocated = freeBlocks.take(numBlocks)
        RETURN allocated

    FUNCTION free(blocks : List<Int>) -> Void
        freeBlocks.addAll(blocks)

    FUNCTION getUtilization() -> Float
        RETURN 1.0 - (freeBlocks.size() / totalBlocks)

    // Prefix caching — Share KV cache blocks for common prefixes
    FUNCTION findSharedPrefix(newTokens : List<Token>) -> SharedPrefix
        FOR EACH existingRequest IN blockTable.keys()
            sharedLength = commonPrefixLength(
                existingRequest.tokens, newTokens
            )
            IF sharedLength >= blockSize THEN
                sharedBlocks = existingRequest.blocks[:sharedLength / blockSize]
                RETURN SharedPrefix(blocks = sharedBlocks, length = sharedLength)
            END IF
        END FOR
        RETURN SharedPrefix.NONE
END CLASS
```

### Model Quantization

Reduce model precision to decrease memory usage and increase inference speed:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Quantization Comparison                           │
├──────────────────┬──────────┬──────────┬───────────────────────────┤
│  Format          │  Bits    │  Memory  │  Notes                    │
├──────────────────┼──────────┼──────────┼───────────────────────────┤
│  FP32            │  32      │  1.0x    │  Full precision (training)│
│  FP16 / BF16     │  16      │  0.5x   │  Standard inference       │
│  INT8 (W8A8)     │  8       │  0.25x  │  Good quality/speed       │
│  INT4 (GPTQ/AWQ) │  4       │  0.125x │  Slight quality loss      │
│  GGUF (2-6 bit)  │  2-6     │  varies │  CPU-friendly, flexible   │
│  FP8 (E4M3)      │  8       │  0.25x  │  Hardware-native on H100  │
└──────────────────┴──────────┴──────────┴───────────────────────────┘
```

```
CLASS QuantizationPipeline
    PROPERTIES
        calibrationData : List<String>      // Representative prompts
        targetFormat    : QuantFormat

    FUNCTION quantize(model : FullPrecisionModel) -> QuantizedModel
        // 1. Run calibration data through model to collect activation statistics
        activationStats = collectActivationStatistics(model, calibrationData)

        // 2. Compute quantization parameters per layer
        quantParams = computeScalesAndZeroPoints(activationStats, targetFormat)

        // 3. Quantize weights
        quantizedWeights = quantizeWeights(model.weights, quantParams)

        // 4. Validate quality
        originalPerplexity = evaluatePerplexity(model, calibrationData)
        quantizedPerplexity = evaluatePerplexity(quantizedWeights, calibrationData)

        IF (quantizedPerplexity - originalPerplexity) > QUALITY_THRESHOLD THEN
            LOG.WARN("Quality degradation exceeds threshold: " +
                     quantizedPerplexity + " vs " + originalPerplexity)
        END IF

        RETURN QuantizedModel(weights = quantizedWeights, format = targetFormat)
END CLASS
```

### Speculative Decoding

Use a small, fast "draft" model to generate candidate tokens, then verify them in parallel with the large model:

```
CLASS SpeculativeDecoder
    PROPERTIES
        targetModel  : LargeModel       // Slow, high-quality
        draftModel   : SmallModel       // Fast, approximate
        numSpecTokens: Int = 5          // Tokens to speculate per step

    FUNCTION generate(prompt : List<Token>, maxTokens : Int) -> List<Token>
        generated = EMPTY LIST

        WHILE generated.length < maxTokens
            // 1. Draft model generates K candidate tokens autoregressively
            draftTokens = draftModel.generateTokens(
                prompt + generated, count = numSpecTokens
            )

            // 2. Target model verifies ALL draft tokens in ONE forward pass
            //    (this is the key insight — verification is parallelizable)
            targetLogits = targetModel.forward(prompt + generated + draftTokens)

            // 3. Accept tokens that match target model's distribution
            accepted = 0
            FOR i IN 0..numSpecTokens
                IF passesAcceptanceCriterion(draftTokens[i], targetLogits[i]) THEN
                    generated.ADD(draftTokens[i])
                    accepted += 1
                ELSE
                    // Sample correct token from target model at rejection point
                    correctedToken = sampleFromAdjusted(
                        targetLogits[i], draftLogits[i]
                    )
                    generated.ADD(correctedToken)
                    BREAK
                END IF
            END FOR

            // Acceptance rate determines speedup (typically 60-80%)
        END WHILE

        RETURN generated
END CLASS
```

### Autoscaling

Scale inference infrastructure based on demand:

```
CLASS InferenceAutoscaler
    PROPERTIES
        minReplicas     : Int
        maxReplicas     : Int
        metrics         : MetricsCollector
        scalingPolicy   : ScalingPolicy

    FUNCTION evaluate() -> ScalingDecision
        currentMetrics = metrics.collect()

        // Primary scaling signals
        queueDepth     = currentMetrics.requestQueueDepth
        gpuUtilization = currentMetrics.avgGPUUtilization
        p99Latency     = currentMetrics.p99Latency
        currentReplicas = currentMetrics.replicaCount

        IF queueDepth > scalingPolicy.queueThreshold
           OR p99Latency > scalingPolicy.latencyThreshold THEN
            newReplicas = MIN(currentReplicas + scalingPolicy.scaleUpStep,
                              maxReplicas)
            RETURN ScalingDecision(action = SCALE_UP, replicas = newReplicas)

        ELSE IF gpuUtilization < scalingPolicy.scaleDownUtilization
                AND queueDepth == 0
                AND currentReplicas > minReplicas THEN
            newReplicas = MAX(currentReplicas - 1, minReplicas)
            RETURN ScalingDecision(action = SCALE_DOWN, replicas = newReplicas)
        END IF

        RETURN ScalingDecision(action = NO_CHANGE)
END CLASS
```

### Model Deployment Strategies

```
┌─────────────────────────────────────────────────────────────────────┐
│                  Deployment Strategies                                │
├──────────────────┬──────────────────────────────────────────────────┤
│  Strategy        │  Description                                     │
├──────────────────┼──────────────────────────────────────────────────┤
│  Blue-Green      │  Two full environments. Switch traffic between   │
│                  │  old (blue) and new (green) model versions.      │
│                  │  Fast rollback by switching back.                │
├──────────────────┼──────────────────────────────────────────────────┤
│  Canary          │  Route a small percentage of traffic to the new  │
│                  │  model version. Gradually increase if quality    │
│                  │  metrics hold. Minimize blast radius.            │
├──────────────────┼──────────────────────────────────────────────────┤
│  Shadow          │  Run new model in parallel with production model.│
│                  │  Compare outputs offline without affecting users.│
│                  │  Highest confidence but highest cost.            │
├──────────────────┼──────────────────────────────────────────────────┤
│  A/B Test        │  Split traffic between model versions to measure │
│                  │  business impact. Requires evaluation framework. │
└──────────────────┴──────────────────────────────────────────────────┘
```

## Project Structure

```
src/
├── server/                         # Inference Server
│   ├── api/                        # HTTP/gRPC endpoints
│   ├── scheduler/                  # Batch scheduling
│   │   ├── continuous_batching/
│   │   └── static_batching/
│   ├── executor/                   # Model execution engine
│   └── streamer/                   # Response streaming (SSE)
│
├── memory/                         # GPU Memory Management
│   ├── kv_cache/
│   │   ├── paged_attention/
│   │   └── prefix_cache/
│   └── block_manager/
│
├── optimization/                   # Model Optimization
│   ├── quantization/
│   │   ├── gptq/
│   │   ├── awq/
│   │   └── gguf/
│   ├── speculative_decoding/
│   └── compilation/                # torch.compile, TensorRT
│
├── deployment/                     # Deployment and Scaling
│   ├── model_registry/
│   ├── autoscaler/
│   ├── health_checks/
│   └── strategies/                 # Blue-green, canary, shadow
│
├── observability/                  # Metrics and Monitoring
│   ├── metrics/                    # TTFT, ITL, throughput, GPU util
│   ├── tracing/
│   └── dashboards/
│
├── config/                         # Configuration
│
└── tests/
    ├── unit/
    ├── integration/
    └── benchmarks/                 # Throughput and latency benchmarks
```

## Key Design Considerations

### Latency Budgets

Define clear latency budgets for different use cases:

- **Interactive chat** — TTFT < 500ms, ITL < 50ms
- **Batch processing** — Optimize for throughput (tokens/second), latency less critical
- **Real-time classification** — End-to-end < 100ms
- **Streaming** — Consistent ITL matters more than TTFT

### GPU Memory Planning

Model serving is memory-bound. Plan GPU allocation carefully:

- **Model weights** — Fixed cost based on model size and precision (e.g., 7B FP16 ≈ 14GB)
- **KV cache** — Dynamic, proportional to batch size × sequence length × model dimensions
- **Activation memory** — Temporary during forward pass, freed after each step
- **Reserve headroom** — Keep 10-15% GPU memory free for stability

### Multi-GPU Strategies

For models that exceed single GPU memory:

- **Tensor parallelism** — Split individual layers across GPUs; reduces per-GPU memory, adds inter-GPU communication
- **Pipeline parallelism** — Assign different layers to different GPUs; simpler communication but harder to balance
- **Data parallelism** — Same model on each GPU, split requests; simplest but requires model to fit on one GPU

### Cost Optimization

- **Right-size GPU selection** — Match GPU type to workload (A10G for smaller models, A100/H100 for large)
- **Spot/preemptible instances** — Use for batch inference; not recommended for real-time serving
- **Quantization ROI** — INT4 quantization can halve GPU requirements with minimal quality loss
- **Request batching** — Higher batch sizes amortize per-request overhead dramatically

## Benefits

1. **Low Latency** — Continuous batching and KV-cache optimization minimize response times
2. **High Throughput** — Serve more requests per GPU through efficient batching and scheduling
3. **Cost Efficiency** — Quantization and model routing reduce GPU requirements
4. **Reliability** — Autoscaling, health checks, and deployment strategies ensure uptime
5. **Flexibility** — Support multiple model versions and deployment strategies simultaneously
6. **Observability** — Detailed metrics guide optimization decisions

## Trade-offs

| Advantage                            | Consideration                                            |
| ------------------------------------ | -------------------------------------------------------- |
| Continuous batching throughput gains | More complex scheduling logic                            |
| Quantization reduces memory and cost | Potential quality degradation at aggressive quantization |
| Speculative decoding speedup         | Requires maintaining a separate draft model              |
| Autoscaling handles variable load    | GPU cold-start time (model loading) can be significant   |
| Multi-GPU enables large models       | Inter-GPU communication overhead and complexity          |
| KV-cache prefix sharing              | Memory management complexity increases                   |

## When to Use

✅ **Good fit for:**

- Production LLM deployments serving real-time user traffic
- High-throughput batch inference pipelines (document processing, evaluation)
- Multi-model serving environments with diverse workloads
- Cost-optimized GPU infrastructure for AI applications
- Scenarios requiring consistent SLAs for AI inference

❌ **Not ideal for:**

- Prototyping or development where API-based models (OpenAI, Anthropic) are simpler
- Low-traffic applications where managed API services are more cost-effective
- Models small enough to run on CPU without specialized infrastructure
- Teams without GPU infrastructure expertise

## References

- [vLLM: Easy, Fast, and Cheap LLM Serving — Kwon et al. (2023)](https://arxiv.org/abs/2309.06180)
- [Efficient Memory Management for Large Language Model Serving with PagedAttention — Kwon et al. (2023)](https://arxiv.org/abs/2309.06180)
- [Fast Inference from Transformers via Speculative Decoding — Leviathan et al. (2023)](https://arxiv.org/abs/2211.17192)
- [GPTQ: Accurate Post-Training Quantization — Frantar et al. (2023)](https://arxiv.org/abs/2210.17323)
- [AWQ: Activation-aware Weight Quantization — Lin et al. (2023)](https://arxiv.org/abs/2306.00978)
- [NVIDIA Triton Inference Server](https://developer.nvidia.com/triton-inference-server)
- [Text Generation Inference (TGI) — Hugging Face](https://huggingface.co/docs/text-generation-inference)
- [Orca: A Distributed Serving System for Transformer-Based Models — Yu et al. (2022)](https://www.usenix.org/conference/osdi22/presentation/yu)
