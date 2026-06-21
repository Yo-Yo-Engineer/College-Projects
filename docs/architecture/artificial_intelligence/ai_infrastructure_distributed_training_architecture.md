# AI Infrastructure & Distributed Training Architecture

## Overview

**AI Infrastructure & Distributed Training Architecture** defines the structural patterns for building, training, and operating large-scale AI models across distributed compute clusters. Unlike traditional application infrastructure — where scaling means adding stateless replicas — AI training infrastructure must coordinate GPU/accelerator memory, synchronize gradient updates across nodes, manage massive datasets, and handle checkpoint/recovery for jobs that run for days or weeks.

This document covers distributed training strategies, GPU cluster management, training pipeline orchestration, checkpoint and fault tolerance, and the infrastructure required to go from raw data to a deployed model.

Key principles:

- **Parallelism as Architecture** — Data parallelism, model parallelism, and pipeline parallelism are fundamental design choices, not optimizations
- **Fault Tolerance by Default** — Long-running training jobs must survive hardware failures through checkpointing and automatic recovery
- **Resource Efficiency** — GPU hours are expensive; the architecture must maximize utilization and minimize idle time
- **Reproducibility** — Training runs must be reproducible given the same data, code, configuration, and random seeds
- **Separation of Compute and Storage** — Datasets, checkpoints, and model artifacts live in durable storage decoupled from ephemeral compute

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│           AI Infrastructure & Distributed Training Architecture             │
│                                                                             │
│   ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐   │
│   │  Data Pipeline     │────▶│  Training         │────▶│  Model Registry   │  │
│   │  (prep, tokenize,  │     │  Orchestrator      │     │  (artifacts,      │  │
│   │   shard, validate) │     │  (job scheduler)   │     │   versions)       │  │
│   └──────────────────┘     └────────┬─────────┘     └──────────────────┘   │
│                                      │                                      │
│                         ┌────────────┼────────────┐                         │
│                         │            │            │                         │
│                         ▼            ▼            ▼                         │
│                   ┌──────────┐ ┌──────────┐ ┌──────────┐                   │
│                   │  Node 0   │ │  Node 1   │ │  Node N   │                 │
│                   │  (Rank 0)  │ │  (Rank 1)  │ │  (Rank N) │               │
│                   │ ┌──┬──┐  │ │ ┌──┬──┐  │ │ ┌──┬──┐  │                   │
│                   │ │G0│G1│  │ │ │G0│G1│  │ │ │G0│G1│  │                   │
│                   │ └──┴──┘  │ │ └──┴──┘  │ │ └──┴──┘  │                   │
│                   └─────┬────┘ └─────┬────┘ └─────┬────┘                   │
│                         │            │            │                         │
│                         └────────────┼────────────┘                         │
│                                      │                                      │
│                            ┌─────────▼──────────┐                          │
│                            │  High-Speed          │                         │
│                            │  Interconnect         │                        │
│                            │  (NVLink, InfiniBand) │                        │
│                            └─────────────────────┘                          │
│                                                                             │
│   Storage Layer:                                                            │
│   ┌────────────┐  ┌──────────────┐  ┌────────────┐  ┌──────────────────┐  │
│   │ Dataset     │  │  Checkpoint   │  │  Model      │  │  Experiment       │  │
│   │ Store       │  │  Store        │  │  Artifacts  │  │  Tracker          │  │
│   └────────────┘  └──────────────┘  └────────────┘  └──────────────────┘  │
│                                                                             │
│   Cross-Cutting:                                                            │
│   ┌────────────┐  ┌──────────────┐  ┌────────────┐  ┌──────────────────┐  │
│   │ Cluster     │  │  Job          │  │  GPU         │  │  Cost              │  │
│   │ Manager     │  │  Scheduler    │  │  Monitoring  │  │  Tracking          │  │
│   └────────────┘  └──────────────┘  └────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Core Concepts

### Parallelism Strategies

Distributing training across multiple GPUs/nodes requires choosing the right parallelism strategy:

```
┌───────────────────────────────────────────────────────────────────────┐
│                   Parallelism Strategies                              │
├──────────────────┬────────────────────────────────────────────────────┤
│  Strategy        │  Description                                      │
├──────────────────┼────────────────────────────────────────────────────┤
│  Data            │  Same model replicated across GPUs. Each GPU      │
│  Parallelism     │  processes a different data shard. Gradients      │
│                  │  synchronized via all-reduce. Simplest strategy.  │
├──────────────────┼────────────────────────────────────────────────────┤
│  Model /         │  Model layers split across GPUs. Each GPU holds   │
│  Tensor          │  a portion of the model's parameters. Required    │
│  Parallelism     │  when model exceeds single GPU memory.            │
├──────────────────┼────────────────────────────────────────────────────┤
│  Pipeline        │  Model split into sequential stages across GPUs.  │
│  Parallelism     │  Micro-batches flow through the pipeline.         │
│                  │  Reduces idle time via micro-batch interleaving.  │
├──────────────────┼────────────────────────────────────────────────────┤
│  Sequence        │  Long input sequences split across GPUs. Each     │
│  Parallelism     │  GPU processes a portion of the sequence.         │
│                  │  Critical for very long context training.         │
├──────────────────┼────────────────────────────────────────────────────┤
│  Expert          │  Mixture-of-Experts models route tokens to        │
│  Parallelism     │  different expert subnetworks on different GPUs.  │
│                  │  Scales model capacity without proportional cost. │
├──────────────────┼────────────────────────────────────────────────────┤
│  Fully Sharded   │  Model parameters, gradients, and optimizer       │
│  Data Parallel   │  states sharded across GPUs. Each GPU holds       │
│  (FSDP / ZeRO)   │  only a fraction. Reduces memory per GPU.        │
└──────────────────┴────────────────────────────────────────────────────┘
```

### Parallelism Visualizations

#### Data Parallelism

```
                 ┌── GPU 0: Full Model Copy ──▶ Grad₀ ──┐
                 │                                       │
Data Batch ──▶ Split ── GPU 1: Full Model Copy ──▶ Grad₁ ──┼──▶ All-Reduce ──▶ Updated Model
                 │                                       │
                 └── GPU 2: Full Model Copy ──▶ Grad₂ ──┘
```

#### Pipeline Parallelism

```
GPU 0: [Stage 1]  ▓▓░░▓▓░░▓▓░░          ▓ = compute
GPU 1: [Stage 2]  ░░▓▓░░▓▓░░▓▓          ░ = idle (bubble)
GPU 2: [Stage 3]  ░░░░▓▓░░▓▓░░▓▓
                  ─────────────────▶ Time

Micro-batch 1: GPU0 → GPU1 → GPU2
Micro-batch 2:        GPU0 → GPU1 → GPU2   (overlapped)
```

#### FSDP / ZeRO (Fully Sharded Data Parallelism)

```
┌────────────────────────────────────────────────────────┐
│              FSDP / ZeRO Sharding Stages               │
│                                                        │
│  ZeRO Stage 1: Shard optimizer states only            │
│  ZeRO Stage 2: Shard optimizer states + gradients     │
│  ZeRO Stage 3: Shard optimizer + gradients + params   │
│                                                        │
│  GPU 0          GPU 1          GPU 2                   │
│  ┌────────┐    ┌────────┐    ┌────────┐               │
│  │Params⅓ │    │Params⅓ │    │Params⅓ │               │
│  │Grads⅓  │    │Grads⅓  │    │Grads⅓  │               │
│  │Optim⅓  │    │Optim⅓  │    │Optim⅓  │               │
│  └────────┘    └────────┘    └────────┘               │
│                                                        │
│  All-gather params before forward/backward pass       │
│  Reduce-scatter gradients after backward pass         │
└────────────────────────────────────────────────────────┘
```

### Training Pipeline Stages

```
┌───────────────────────────────────────────────────────────────────────┐
│                   Training Pipeline                                   │
│                                                                       │
│  1. Data Preparation                                                  │
│     Raw Data ──▶ Clean ──▶ Tokenize ──▶ Shard ──▶ Dataset Store     │
│                                                                       │
│  2. Pre-Training                                                      │
│     Dataset ──▶ Distributed Training Loop ──▶ Base Model             │
│                   (self-supervised, next-token prediction)            │
│                                                                       │
│  3. Fine-Tuning                                                       │
│     Base Model ──▶ Task-Specific Training ──▶ Fine-Tuned Model       │
│                   (supervised, instruction tuning, LoRA/QLoRA)       │
│                                                                       │
│  4. Alignment                                                         │
│     Fine-Tuned Model ──▶ RLHF / DPO ──▶ Aligned Model              │
│                   (human preferences, reward modeling)                │
│                                                                       │
│  5. Evaluation & Export                                                │
│     Aligned Model ──▶ Benchmarks ──▶ Quantize ──▶ Model Registry    │
└───────────────────────────────────────────────────────────────────────┘
```

### Memory Optimization Techniques

```
┌───────────────────────────────────────────────────────────────────────┐
│                   GPU Memory Optimization                            │
├──────────────────┬────────────────────────────────────────────────────┤
│  Technique       │  Description                                      │
├──────────────────┼────────────────────────────────────────────────────┤
│  Mixed Precision │  Train with FP16/BF16 for compute, FP32 for      │
│  Training        │  master weights. ~2x memory savings, faster math. │
├──────────────────┼────────────────────────────────────────────────────┤
│  Gradient        │  Accumulate gradients over multiple micro-batches │
│  Accumulation    │  before updating weights. Simulates larger batch  │
│                  │  sizes without proportional GPU memory.           │
├──────────────────┼────────────────────────────────────────────────────┤
│  Activation      │  Recompute activations during backward pass       │
│  Checkpointing   │  instead of storing them. Trades compute for      │
│                  │  memory. ~30-40% memory reduction.                │
├──────────────────┼────────────────────────────────────────────────────┤
│  Gradient        │  Offload optimizer states and gradients to CPU    │
│  Offloading      │  RAM or NVMe. Slower but enables larger models.  │
├──────────────────┼────────────────────────────────────────────────────┤
│  Flash           │  Memory-efficient attention that avoids           │
│  Attention       │  materializing the full attention matrix.         │
│                  │  O(N) memory instead of O(N²).                    │
├──────────────────┼────────────────────────────────────────────────────┤
│  Quantized       │  QLoRA: 4-bit quantized base model with FP16     │
│  Training        │  LoRA adapters. Enables fine-tuning large models  │
│                  │  on consumer GPUs.                                │
└──────────────────┴────────────────────────────────────────────────────┘
```

## Implementation

### Distributed Training Configuration

```
// Training Configuration — Defines a distributed training job
DATA TrainingConfig
    // Model
    modelArchitecture   : String          // "transformer", "mixture_of_experts"
    modelSize           : String          // "7B", "70B", "405B"
    pretrainedCheckpoint : String OR NULL // Resume from checkpoint

    // Data
    datasetPaths        : List<String>
    tokenizer           : String
    maxSeqLength        : Integer
    dataParallelShards  : Integer

    // Distributed Strategy
    parallelismConfig   : ParallelismConfig

    // Optimization
    optimizer           : OptimizerConfig
    scheduler           : LRSchedulerConfig
    mixedPrecision      : String          // "fp16", "bf16", "fp32"
    gradientAccumSteps  : Integer
    activationCheckpointing : Boolean

    // Training
    totalSteps          : Integer
    batchSizePerGPU     : Integer
    warmupSteps         : Integer

    // Checkpointing
    checkpointInterval  : Integer         // Save every N steps
    checkpointPath      : String
    maxCheckpointsKept  : Integer

    // Logging
    logInterval         : Integer
    experimentTracker   : String          // "wandb", "mlflow", "tensorboard"
END DATA

DATA ParallelismConfig
    dataParallelSize     : Integer        // Number of data parallel replicas
    tensorParallelSize   : Integer        // Tensor parallelism degree
    pipelineParallelSize : Integer        // Pipeline parallelism stages
    expertParallelSize   : Integer        // Expert parallelism (MoE)
    fsdpEnabled          : Boolean        // Fully Sharded Data Parallelism
    fsdpShardingStrategy : String         // "full_shard", "shard_grad_op", "no_shard"
END DATA
```

### Distributed Training Loop

```
// Distributed Training Loop — Core training execution
CLASS DistributedTrainer
    CONSTRUCTOR(
        config          : TrainingConfig,
        model           : DistributedModel,
        optimizer       : Optimizer,
        dataLoader      : DistributedDataLoader,
        checkpointMgr   : CheckpointManager,
        experimentTracker : ExperimentTracker,
        communicator    : DistributedCommunicator
    )

    FUNCTION train() -> TrainingResult
        rank = communicator.getRank()
        worldSize = communicator.getWorldSize()

        // Resume from checkpoint if available
        startStep = 0
        IF config.pretrainedCheckpoint IS NOT NULL THEN
            startStep = checkpointMgr.load(
                path      = config.pretrainedCheckpoint,
                model     = model,
                optimizer = optimizer
            )
            LOG("Resumed from step " + startStep + " on rank " + rank)
        END IF

        model.setTrainingMode(TRUE)

        FOR step = startStep TO config.totalSteps
            // 1. Get next batch (each rank gets a different shard)
            batch = dataLoader.getNextBatch(rank)

            // 2. Forward pass (with mixed precision)
            WITH mixedPrecisionContext(config.mixedPrecision)
                loss = model.forward(batch)
                scaledLoss = loss / config.gradientAccumSteps
            END WITH

            // 3. Backward pass
            scaledLoss.backward()

            // 4. Gradient accumulation
            IF (step + 1) % config.gradientAccumSteps == 0 THEN
                // Gradient clipping
                gradNorm = clipGradientNorm(model.parameters(), maxNorm = 1.0)

                // Optimizer step
                optimizer.step()
                optimizer.zeroGrad()
                scheduler.step()
            END IF

            // 5. Logging (rank 0 only)
            IF rank == 0 AND step % config.logInterval == 0 THEN
                experimentTracker.log({
                    step          : step,
                    loss          : loss.item(),
                    learningRate  : scheduler.getCurrentLR(),
                    gradNorm      : gradNorm,
                    tokensPerSec  : calculateThroughput(batch, stepTime),
                    gpuMemoryUsed : getGPUMemoryUsage(),
                    gpuUtilization : getGPUUtilization()
                })
            END IF

            // 6. Checkpointing
            IF step % config.checkpointInterval == 0 THEN
                checkpointMgr.save(
                    path      = config.checkpointPath + "/step_" + step,
                    model     = model,
                    optimizer = optimizer,
                    step      = step
                )
            END IF

        END FOR

        RETURN NEW TrainingResult(
            finalLoss    = loss.item(),
            totalSteps   = config.totalSteps,
            checkpoints  = checkpointMgr.listCheckpoints()
        )
END CLASS
```

### Checkpoint Manager

```
// Checkpoint Manager — Save and restore training state
CLASS CheckpointManager
    CONSTRUCTOR(
        storage      : DistributedStorage,
        communicator : DistributedCommunicator
    )

    FUNCTION save(path : String, model : DistributedModel,
                  optimizer : Optimizer, step : Integer) -> Void
        rank = communicator.getRank()

        // Each rank saves its shard of the model/optimizer state
        shardPath = path + "/rank_" + rank

        state = {
            modelState     : model.getShardedState(rank),
            optimizerState : optimizer.getShardedState(rank),
            step           : step,
            rngState       : getRNGState(),
            worldSize      : communicator.getWorldSize()
        }

        storage.save(shardPath, state)

        // Rank 0 saves metadata
        IF rank == 0 THEN
            metadata = {
                step            : step,
                worldSize       : communicator.getWorldSize(),
                parallelismConfig : config.parallelismConfig,
                timestamp       : NOW()
            }
            storage.save(path + "/metadata.json", metadata)
        END IF

        // Barrier: ensure all ranks have saved before proceeding
        communicator.barrier()

        // Cleanup old checkpoints
        IF rank == 0 THEN
            cleanupOldCheckpoints(config.maxCheckpointsKept)
        END IF

    FUNCTION load(path : String, model : DistributedModel,
                  optimizer : Optimizer) -> Integer
        rank = communicator.getRank()
        metadata = storage.load(path + "/metadata.json")

        // Handle world size changes (elastic training)
        IF metadata.worldSize != communicator.getWorldSize() THEN
            state = loadAndReshardState(path, metadata, rank)
        ELSE
            state = storage.load(path + "/rank_" + rank)
        END IF

        model.loadShardedState(state.modelState, rank)
        optimizer.loadShardedState(state.optimizerState, rank)
        setRNGState(state.rngState)

        communicator.barrier()
        RETURN state.step
END CLASS
```

### Data Pipeline

```
// Distributed Data Pipeline — Prepare and shard data across ranks
CLASS DistributedDataPipeline
    CONSTRUCTOR(
        tokenizer    : Tokenizer,
        storage      : DistributedStorage,
        config       : TrainingConfig
    )

    FUNCTION prepare(rawDataPaths : List<String>) -> ShardedDataset
        // 1. Load and clean raw data
        documents = EMPTY LIST
        FOR EACH path IN rawDataPaths
            rawData = storage.load(path)
            cleaned = clean(rawData)          // dedup, filter, normalize
            documents.APPEND_ALL(cleaned)
        END FOR

        // 2. Tokenize
        tokenized = EMPTY LIST
        FOR EACH doc IN documents
            tokens = tokenizer.encode(doc)
            tokenized.ADD(tokens)
        END FOR

        // 3. Pack into sequences of maxSeqLength
        sequences = packSequences(tokenized, config.maxSeqLength)

        // 4. Shuffle and shard
        sequences.SHUFFLE(seed = config.dataSeed)
        shards = splitIntoShards(sequences, config.dataParallelShards)

        // 5. Save shards to storage
        FOR i = 0 TO shards.SIZE() - 1
            storage.save("shards/shard_" + i, shards[i])
        END FOR

        RETURN NEW ShardedDataset(
            numShards    = shards.SIZE(),
            numSequences = sequences.SIZE(),
            totalTokens  = SUM(s.SIZE() FOR s IN sequences)
        )
END CLASS

// Distributed Data Loader — Feeds shards to the correct GPU
CLASS DistributedDataLoader
    CONSTRUCTOR(
        dataset      : ShardedDataset,
        batchSize    : Integer,
        worldSize    : Integer,
        numWorkers   : Integer
    )

    FUNCTION getNextBatch(rank : Integer) -> Batch
        // Each rank reads from its assigned shard(s)
        shardIndex = rank % dataset.numShards
        batch = readBatchFromShard(shardIndex, batchSize)
        RETURN batch.toGPU(rank)
END CLASS
```

### Cluster Manager

```
// Cluster Manager — Manage GPU nodes for training jobs
CLASS ClusterManager
    CONSTRUCTOR(
        scheduler      : JobScheduler,
        nodePool       : NodePool,
        healthChecker  : HealthChecker,
        costTracker    : CostTracker
    )

    FUNCTION submitJob(jobConfig : TrainingJobConfig) -> Job
        // 1. Calculate resource requirements
        requiredGPUs = (
            jobConfig.parallelism.dataParallelSize *
            jobConfig.parallelism.tensorParallelSize *
            jobConfig.parallelism.pipelineParallelSize
        )

        // 2. Request nodes from pool
        nodes = nodePool.allocate(
            numGPUs        = requiredGPUs,
            gpuType        = jobConfig.gpuType,  // "A100", "H100"
            interconnect   = jobConfig.interconnect  // "NVLink", "InfiniBand"
        )

        IF nodes IS NULL THEN
            RETURN scheduler.enqueue(jobConfig)  // Queue for later
        END IF

        // 3. Launch distributed training
        job = NEW Job(
            id     = generateId(),
            config = jobConfig,
            nodes  = nodes,
            status = "RUNNING"
        )

        launchDistributed(job)
        RETURN job

    FUNCTION handleNodeFailure(job : Job, failedNode : Node) -> Void
        // 1. Save checkpoint on healthy nodes
        signalCheckpoint(job)

        // 2. Replace failed node
        replacementNode = nodePool.allocate(
            numGPUs   = failedNode.numGPUs,
            gpuType   = failedNode.gpuType
        )

        IF replacementNode IS NOT NULL THEN
            // 3. Resume from latest checkpoint with new node
            job.replaceNode(failedNode, replacementNode)
            resumeFromCheckpoint(job)
        ELSE
            // 4. Reduce world size (elastic training) or pause
            IF job.config.elasticTraining THEN
                job.reduceWorldSize(failedNode)
                resumeFromCheckpoint(job)
            ELSE
                job.status = "PAUSED_NODE_FAILURE"
            END IF
        END IF
END CLASS
```

### Experiment Tracker

```
// Experiment Tracker — Record and compare training runs
CLASS ExperimentTracker
    CONSTRUCTOR(backend : TrackingBackend)

    FUNCTION createRun(config : TrainingConfig) -> RunId
        run = backend.createRun(
            name   = config.experimentName,
            config = serialize(config),
            tags   = config.tags
        )
        RETURN run.id

    FUNCTION log(metrics : Map<String, Any>) -> Void
        backend.logMetrics(currentRunId, metrics)

    FUNCTION logArtifact(name : String, path : String) -> Void
        backend.logArtifact(currentRunId, name, path)

    FUNCTION compareRuns(runIds : List<RunId>,
                         metrics : List<String>) -> ComparisonReport
        data = EMPTY LIST
        FOR EACH runId IN runIds
            runMetrics = backend.getMetrics(runId, metrics)
            runConfig  = backend.getConfig(runId)
            data.ADD({ runId: runId, metrics: runMetrics, config: runConfig })
        END FOR

        RETURN NEW ComparisonReport(runs = data)
END CLASS
```

## Project Structure

```
src/
├── training/                          # Training Core
│   ├── loop/                          # Distributed training loop
│   ├── config/                        # Training configurations
│   ├── optimizers/                    # Custom optimizers (AdamW, LAMB)
│   ├── schedulers/                    # Learning rate schedulers
│   └── losses/                        # Loss functions
│
├── parallelism/                       # Parallelism Strategies
│   ├── data_parallel/                 # DDP, FSDP implementations
│   ├── tensor_parallel/              # Megatron-style tensor parallelism
│   ├── pipeline_parallel/             # Pipeline stage management
│   ├── sequence_parallel/             # Long-sequence parallelism
│   └── expert_parallel/              # MoE expert distribution
│
├── data/                              # Data Pipeline
│   ├── preparation/                   # Cleaning, deduplication, filtering
│   ├── tokenization/                  # Tokenizer training and encoding
│   ├── sharding/                      # Dataset sharding across nodes
│   └── loader/                        # Distributed data loading
│
├── checkpoint/                        # Checkpoint Management
│   ├── save/                          # Sharded checkpoint saving
│   ├── load/                          # Checkpoint loading and resharding
│   └── cleanup/                       # Old checkpoint cleanup
│
├── cluster/                           # Cluster Management
│   ├── scheduler/                     # Job scheduling (FIFO, priority, fair-share)
│   ├── nodes/                         # Node pool management
│   ├── health/                        # Health checking and failure detection
│   └── elastic/                       # Elastic training (dynamic world size)
│
├── memory_optimization/               # GPU Memory Optimization
│   ├── mixed_precision/               # FP16/BF16 training
│   ├── activation_checkpointing/      # Gradient checkpointing
│   ├── gradient_offloading/           # CPU/NVMe offloading
│   └── flash_attention/               # Memory-efficient attention
│
├── experiment_tracking/               # Experiment Management
│   ├── tracker/                       # Metrics logging
│   ├── comparison/                    # Run comparison and analysis
│   └── artifacts/                     # Model artifact management
│
├── evaluation/                        # Model Evaluation
│   ├── benchmarks/                    # Standard benchmarks (MMLU, HumanEval, etc.)
│   ├── custom/                        # Custom evaluation suites
│   └── reports/                       # Evaluation reports
│
├── observability/                     # Monitoring
│   ├── gpu_metrics/                   # GPU utilization, memory, temperature
│   ├── training_metrics/              # Loss curves, throughput, gradient norms
│   ├── cluster_metrics/               # Node health, network throughput
│   └── alerts/                        # Anomaly detection and alerting
│
├── config/
│
└── tests/
    ├── unit/
    ├── integration/
    └── distributed/                   # Multi-GPU/multi-node tests
```

## Benefits

1. **Scale** — Train models from millions to trillions of parameters across hundreds of GPUs
2. **Fault Tolerance** — Automatic checkpointing and recovery ensures long training runs complete despite hardware failures
3. **Resource Efficiency** — Mixed precision, activation checkpointing, and FSDP maximize GPU memory utilization
4. **Reproducibility** — Deterministic data sharding, seeded RNG, and experiment tracking enable exact reproduction of training runs
5. **Flexibility** — Composable parallelism strategies adapt to different model sizes and cluster configurations
6. **Visibility** — Real-time monitoring of GPU utilization, training metrics, and cluster health

## Trade-offs

| Advantage                              | Consideration                                        |
| -------------------------------------- | ---------------------------------------------------- |
| Multi-node distributed training        | High-speed interconnect (InfiniBand/NVLink) required |
| FSDP reduces per-GPU memory            | All-gather communication adds overhead               |
| Pipeline parallelism overlaps compute  | Pipeline bubbles reduce GPU utilization              |
| Elastic training handles node failures | Resharding checkpoints is complex                    |
| Mixed precision halves memory          | Numerical instability in some model architectures    |
| Activation checkpointing saves memory  | ~30% increase in compute time due to recomputation   |

## When to Use

✅ **Good fit for:**

- Pre-training or fine-tuning models that exceed single GPU memory (> 7B parameters)
- Organizations training proprietary foundation models
- Research teams requiring reproducible, large-scale training experiments
- Production fine-tuning pipelines with multiple concurrent training jobs
- Teams operating GPU clusters (on-premises or cloud)

❌ **Not ideal for:**

- Fine-tuning small models (< 1B parameters) that fit on a single GPU
- Using pre-trained models via API (no training involved)
- Prototyping where a notebook with a single GPU suffices
- Teams without access to multi-GPU infrastructure

## References

- [DeepSpeed: Deep Learning Optimization Library — Microsoft](https://github.com/microsoft/DeepSpeed)
- [Megatron-LM: Training Multi-Billion Parameter Language Models — NVIDIA](https://github.com/NVIDIA/Megatron-LM)
- [FSDP: Fully Sharded Data Parallel — PyTorch](https://pytorch.org/docs/stable/fsdp.html)
- [ZeRO: Memory Optimizations Toward Training Trillion Parameter Models — Microsoft Research (2020)](https://arxiv.org/abs/1910.02054)
- [FlashAttention: Fast and Memory-Efficient Exact Attention — Dao et al. (2022)](https://arxiv.org/abs/2205.14135)
- [Scaling Language Models: Methods, Analysis & Insights from Training Gopher — DeepMind (2022)](https://arxiv.org/abs/2112.11446)
- [Training Compute-Optimal Large Language Models — Hoffmann et al. (Chinchilla, 2022)](https://arxiv.org/abs/2203.15556)
- [Llama 2: Open Foundation and Fine-Tuned Chat Models — Meta (2023)](https://arxiv.org/abs/2307.09288)
