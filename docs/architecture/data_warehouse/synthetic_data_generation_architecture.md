# Synthetic Data Generation Architecture

## Overview

**Synthetic Data Generation Architecture** defines the structural patterns for creating artificial datasets that mimic the statistical properties and structure of real-world data without containing actual real records. As AI systems require ever-larger, more diverse, and more specialized training datasets — while privacy regulations, data scarcity, and labeling costs create constraints — synthetic data has emerged as a critical capability for modern ML pipelines.

Synthetic data is used across the AI lifecycle: generating training data for rare events, augmenting underrepresented classes, creating evaluation datasets, testing data pipelines, enabling privacy-preserving data sharing, and producing instruction-tuning datasets for LLMs.

Key principles:

- **Fidelity** — Synthetic data must be statistically faithful to the real distribution to be useful for training
- **Privacy Guarantee** — Synthetic data should provide formal or practical privacy guarantees, not merely remove identifiers
- **Diversity** — Generated data should cover the full distribution, including edge cases and rare events, not just the common cases
- **Controllability** — Generation should be steerable to produce data with specific characteristics, distributions, or labels
- **Validation** — Always measure synthetic data quality against real data before using it for training or evaluation

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   Synthetic Data Generation Architecture                     │
│                                                                             │
│   ┌──────────────────────────────────────────────────────────────────┐      │
│   │                    Data Specification                             │      │
│   │   ┌──────────┐  ┌──────────────┐  ┌─────────────────────────┐  │      │
│   │   │  Schema   │  │  Distribution │  │  Constraints &          │  │      │
│   │   │  Design   │  │  Config       │  │  Business Rules        │  │      │
│   │   └──────────┘  └──────────────┘  └─────────────────────────┘  │      │
│   └──────────────────────────┬───────────────────────────────────────┘      │
│                              │                                              │
│                              ▼                                              │
│   ┌──────────────────────────────────────────────────────────────────┐      │
│   │                    Generation Engine                              │      │
│   │                                                                  │      │
│   │   ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐  │      │
│   │   │  Statistical  │  │  Deep         │  │  LLM-Based         │  │      │
│   │   │  Models       │  │  Generative   │  │  Generation        │  │      │
│   │   │  (Copulas,    │  │  Models       │  │  (Instruction      │  │      │
│   │   │   Bayesian)   │  │  (GAN, VAE,   │  │   tuning data,    │  │      │
│   │   │              │  │   Diffusion)  │  │   personas)        │  │      │
│   │   └──────────────┘  └──────────────┘  └────────────────────┘  │      │
│   └──────────────────────────┬───────────────────────────────────────┘      │
│                              │                                              │
│                              ▼                                              │
│   ┌──────────────────────────────────────────────────────────────────┐      │
│   │                    Validation & Quality                           │      │
│   │   ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐  │      │
│   │   │  Statistical  │  │  Privacy      │  │  Downstream        │  │      │
│   │   │  Fidelity     │  │  Assessment   │  │  Utility Test     │  │      │
│   │   └──────────────┘  └──────────────┘  └────────────────────┘  │      │
│   └──────────────────────────┬───────────────────────────────────────┘      │
│                              │                                              │
│                              ▼                                              │
│                    Validated Synthetic Dataset                               │
│                                                                             │
│   Cross-Cutting: ┌────────────┐  ┌──────────────┐  ┌────────────────────┐  │
│                   │ Lineage &   │  │  Version      │  │  Access            │  │
│                   │ Provenance  │  │  Control      │  │  Control           │  │
│                   └────────────┘  └──────────────┘  └────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Core Concepts

### Generation Approaches

```
┌─────────────────────────────────────────────────────────────────────┐
│                 Synthetic Data Generation Methods                    │
├──────────────────┬──────────────────────────────────────────────────┤
│  Method          │  Description and Use Cases                       │
├──────────────────┼──────────────────────────────────────────────────┤
│  Statistical     │  Model column distributions, correlations, and  │
│  (Copulas,       │  dependencies. Generate rows that preserve      │
│   Bayesian       │  statistical properties. Best for tabular data. │
│   Networks)      │                                                  │
├──────────────────┼──────────────────────────────────────────────────┤
│  GAN-Based       │  Generator-discriminator pairs learn to produce │
│  (CTGAN,         │  realistic samples. Good for complex tabular    │
│   TimeGAN)       │  and time-series data.                          │
├──────────────────┼──────────────────────────────────────────────────┤
│  VAE-Based       │  Variational autoencoders learn a latent space  │
│  (TVAE)          │  and sample from it. Smooth interpolation       │
│                  │  between data points.                           │
├──────────────────┼──────────────────────────────────────────────────┤
│  Diffusion       │  Denoising diffusion models for high-fidelity   │
│  Models          │  image, audio, and tabular generation.           │
├──────────────────┼──────────────────────────────────────────────────┤
│  LLM-Based       │  Use large language models to generate text     │
│                  │  data: instruction-response pairs, personas,    │
│                  │  conversations, structured records.             │
├──────────────────┼──────────────────────────────────────────────────┤
│  Rule-Based /    │  Programmatic generation using schemas, Faker   │
│  Template        │  libraries, and business rules. Deterministic   │
│                  │  and controllable but low diversity.            │
├──────────────────┼──────────────────────────────────────────────────┤
│  Simulation      │  Physics-based or agent-based simulation to     │
│                  │  generate data for autonomous driving, robotics,│
│                  │  or financial markets.                          │
└──────────────────┴──────────────────────────────────────────────────┘
```

### Tabular Data Generation

Generate synthetic tabular data preserving column distributions and inter-column relationships:

```
CLASS TabularDataGenerator
    PROPERTIES
        model       : GenerativeModel       // CTGAN, Copula, Bayesian Network
        metadata    : TableMetadata         // Column types, constraints

    FUNCTION fit(realData : DataFrame) -> Void
        // 1. Analyze data schema
        metadata = analyzeSchema(realData)

        // 2. Preprocess — handle categorical, numerical, datetime columns
        preprocessed = preprocess(realData, metadata)

        // 3. Train generative model
        model.train(preprocessed, metadata)

    FUNCTION generate(numRows : Int,
                      conditions : Map<String, Any> = {}) -> DataFrame
        // Generate with optional conditioning
        IF conditions.isNotEmpty() THEN
            synthetic = model.conditionalSample(numRows, conditions)
        ELSE
            synthetic = model.sample(numRows)
        END IF

        // Postprocess — reverse preprocessing transforms
        synthetic = postprocess(synthetic, metadata)

        // Enforce constraints
        synthetic = enforceConstraints(synthetic, metadata.constraints)

        RETURN synthetic
END CLASS

DATA TableMetadata
    columns       : List<ColumnSpec>
    constraints   : List<Constraint>        // Unique, range, foreign key, etc.
    primaryKey    : String
    relationships : List<TableRelationship> // Multi-table references

DATA ColumnSpec
    name          : String
    dataType      : DataType               // INT, FLOAT, CATEGORICAL, DATETIME, TEXT
    nullable      : Boolean
    distribution  : DistributionStats       // Mean, std, min, max, categories
END DATA
```

### LLM-Based Text Generation

Use LLMs to generate diverse text datasets for training and evaluation:

```
CLASS LLMDataGenerator
    PROPERTIES
        llm             : LanguageModel
        seedExamples    : List<Example>     // Few-shot examples for style/format

    // Generate instruction-tuning data
    FUNCTION generateInstructionData(
        topic : String,
        numExamples : Int,
        difficulty : DifficultyLevel
    ) -> List<InstructionExample>
        results = EMPTY LIST

        FOR i IN 1..numExamples
            prompt = buildInstructionPrompt(
                topic = topic,
                difficulty = difficulty,
                seedExamples = sampleExamples(seedExamples, k = 3),
                previouslyGenerated = results.takeLast(2)   // Avoid repetition
            )
            response = llm.generate(prompt, temperature = 0.9)
            parsed = parseInstructionResponse(response)

            IF passesQualityFilter(parsed) THEN
                results.ADD(parsed)
            END IF
        END FOR

        // Deduplicate
        results = deduplicateBySimilarity(results, threshold = 0.85)
        RETURN results

    // Generate persona-based conversational data
    FUNCTION generateConversationData(
        personas : List<Persona>,
        scenario : String,
        numConversations : Int
    ) -> List<Conversation>
        results = EMPTY LIST

        FOR i IN 1..numConversations
            persona = personas.random()
            prompt = buildConversationPrompt(persona, scenario)

            conversation = EMPTY LIST
            FOR turn IN 1..MAX_TURNS
                userMessage = llm.generate(userPrompt(persona, conversation))
                assistantMessage = llm.generate(assistantPrompt(conversation + [userMessage]))
                conversation.ADD(Message(role = USER, content = userMessage))
                conversation.ADD(Message(role = ASSISTANT, content = assistantMessage))

                IF conversationComplete(conversation) THEN BREAK
            END FOR

            results.ADD(Conversation(persona, conversation))
        END FOR

        RETURN results
END CLASS
```

### Multi-Table Relational Data

Generate synthetic data that preserves referential integrity across related tables:

```
CLASS RelationalDataGenerator
    PROPERTIES
        tableGenerators : Map<String, TabularDataGenerator>
        relationships   : List<ForeignKeyRelationship>

    FUNCTION fit(database : Map<String, DataFrame>) -> Void
        // 1. Discover relationships from schema
        relationships = discoverRelationships(database)

        // 2. Topologically sort tables (parents before children)
        orderedTables = topologicalSort(database.keys(), relationships)

        // 3. Train generators in dependency order
        FOR EACH tableName IN orderedTables
            generator = TabularDataGenerator(model = selectModel(tableName))
            generator.fit(database[tableName])
            tableGenerators[tableName] = generator
        END FOR

    FUNCTION generate(scaleFactor : Float = 1.0) -> Map<String, DataFrame>
        result = Map<String, DataFrame>()
        orderedTables = topologicalSort(tableGenerators.keys(), relationships)

        FOR EACH tableName IN orderedTables
            generator = tableGenerators[tableName]
            numRows = ROUND(originalRowCount(tableName) * scaleFactor)

            // Parent tables: generate independently
            // Child tables: condition on parent foreign keys
            parentConditions = getParentKeyValues(tableName, result, relationships)
            synthetic = generator.generate(numRows, conditions = parentConditions)

            // Ensure referential integrity
            synthetic = enforceReferentialIntegrity(synthetic, tableName,
                                                    result, relationships)
            result[tableName] = synthetic
        END FOR

        RETURN result
END CLASS
```

### Data Augmentation

Expand existing datasets by creating variations of real examples:

```
CLASS DataAugmentor
    PROPERTIES
        strategies : List<AugmentationStrategy>

    FUNCTION augment(dataset : Dataset,
                     multiplier : Int = 5) -> Dataset
        augmented = dataset.copy()

        FOR EACH example IN dataset
            FOR i IN 1..multiplier
                strategy = strategies.random()
                augmentedExample = strategy.apply(example)
                IF passesQualityCheck(augmentedExample) THEN
                    augmented.ADD(augmentedExample)
                END IF
            END FOR
        END FOR

        RETURN augmented
END CLASS

// Text augmentation strategies
CLASS TextAugmentation IMPLEMENTS AugmentationStrategy
    FUNCTION apply(example : TextExample) -> TextExample
        technique = randomChoice([
            SYNONYM_REPLACEMENT,    // Replace words with synonyms
            BACK_TRANSLATION,       // Translate to another language and back
            PARAPHRASE_LLM,         // Use an LLM to rephrase
            ENTITY_SUBSTITUTION,    // Swap named entities
            SENTENCE_SHUFFLING      // Reorder sentences (where order doesn't matter)
        ])
        RETURN applyTechnique(example, technique)
END CLASS

// Image augmentation strategies
CLASS ImageAugmentation IMPLEMENTS AugmentationStrategy
    FUNCTION apply(example : ImageExample) -> ImageExample
        transforms = randomSubset([
            HORIZONTAL_FLIP,
            ROTATION(angle = random(-15, 15)),
            COLOR_JITTER(brightness = 0.2, contrast = 0.2),
            RANDOM_CROP_AND_RESIZE,
            CUTOUT(numHoles = 2, holeSize = 16),
            GAUSSIAN_NOISE(sigma = 0.01)
        ])
        RETURN applyTransforms(example, transforms)
END CLASS
```

### Quality Validation

Validate that synthetic data is faithful to the original distribution and useful for downstream tasks:

```
CLASS SyntheticDataValidator
    PROPERTIES
        realData     : DataFrame
        syntheticData: DataFrame

    FUNCTION validate() -> ValidationReport
        report = ValidationReport()

        // 1. Statistical Fidelity — Column-level distribution similarity
        FOR EACH column IN realData.columns
            IF column.isNumerical THEN
                ksTest = kolmogorovSmirnovTest(realData[column], syntheticData[column])
                report.addColumnMetric(column, "KS-test p-value", ksTest.pValue)
            ELSE
                jsDivergence = jensenShannonDivergence(
                    realData[column].valueCounts(),
                    syntheticData[column].valueCounts()
                )
                report.addColumnMetric(column, "JS-divergence", jsDivergence)
            END IF
        END FOR

        // 2. Correlation Preservation — Pairwise relationships
        realCorr = realData.correlationMatrix()
        synthCorr = syntheticData.correlationMatrix()
        corrDiff = meanAbsoluteDifference(realCorr, synthCorr)
        report.addMetric("Correlation MAE", corrDiff)

        // 3. Downstream Utility — Train on synthetic, test on real
        modelReal = trainClassifier(realData)
        modelSynth = trainClassifier(syntheticData)
        realAccuracy = modelReal.evaluate(testData)
        synthAccuracy = modelSynth.evaluate(testData)  // Same real test set
        report.addMetric("Train-on-synthetic accuracy", synthAccuracy)
        report.addMetric("Train-on-real accuracy", realAccuracy)
        report.addMetric("Utility ratio", synthAccuracy / realAccuracy)

        // 4. Privacy — Nearest neighbor distance ratio
        privacyScore = computeNearestNeighborDistance(syntheticData, realData)
        report.addMetric("Privacy score (DCR)", privacyScore)

        RETURN report
END CLASS
```

### Privacy Assessment

Verify that synthetic data does not memorize or leak real records:

```
CLASS PrivacyAssessment
    PROPERTIES
        realData      : DataFrame
        syntheticData : DataFrame

    FUNCTION assess() -> PrivacyReport
        report = PrivacyReport()

        // 1. Distance to Closest Record (DCR)
        //    Measures if synthetic records are too close to real records
        dcr = computeDCR(syntheticData, realData)
        report.addMetric("Median DCR", dcr.median)
        report.addMetric("Min DCR", dcr.min)
        report.addMetric("Records with DCR < threshold",
                         dcr.countBelow(threshold = 0.05))

        // 2. Membership Inference Attack
        //    Can an attacker determine if a real record was in the training data?
        miaScore = membershipInferenceAttack(syntheticData, realData, holdoutData)
        report.addMetric("MIA AUC", miaScore)
        // AUC close to 0.5 = good (attacker cannot distinguish)

        // 3. Attribute Inference Attack
        //    Can missing attributes be inferred from synthetic data?
        aiaScore = attributeInferenceAttack(syntheticData, realData)
        report.addMetric("AIA accuracy", aiaScore)

        RETURN report
END CLASS
```

## Project Structure

```
src/
├── generators/                     # Generation Engines
│   ├── tabular/
│   │   ├── statistical/            # Copulas, Bayesian Networks
│   │   ├── gan/                    # CTGAN, TimeGAN
│   │   └── vae/                    # TVAE
│   ├── text/
│   │   ├── llm_generator/          # LLM-based text generation
│   │   ├── template_generator/     # Template / Faker based
│   │   └── augmentation/           # Text augmentation
│   ├── image/
│   │   ├── diffusion/              # Diffusion model generation
│   │   └── augmentation/           # Image augmentation
│   └── relational/                 # Multi-table generation
│
├── specification/                  # Data Specification
│   ├── schema/
│   ├── distributions/
│   └── constraints/
│
├── validation/                     # Quality and Privacy
│   ├── fidelity/                   # Statistical fidelity checks
│   ├── utility/                    # Downstream utility testing
│   ├── privacy/                    # Privacy risk assessment
│   └── reports/
│
├── preprocessing/                  # Data Analysis and Prep
│   ├── profiling/                  # Analyze real data distributions
│   ├── encoding/                   # Category encoding, normalization
│   └── relationship_discovery/
│
├── config/                         # Configuration
│
└── tests/
    ├── unit/
    ├── integration/
    └── quality_benchmarks/
```

## Key Design Considerations

### Fidelity vs. Privacy Trade-off

Higher fidelity increases the risk of memorizing real records:

- **Apply differential privacy** during training to provide formal privacy guarantees
- **Monitor DCR (Distance to Closest Record)** — synthetic records too close to real records may leak information
- **Set privacy budgets** and validate with membership inference attacks
- **Remove outliers from training data** — unique records are most vulnerable to memorization

### Conditioning and Controllability

Production use cases often need synthetic data with specific properties:

- **Class balancing** — Generate more examples of underrepresented classes
- **Edge case generation** — Produce rare events (fraud, equipment failure) for training
- **Scenario generation** — Create data matching specific business scenarios (high-traffic periods, market crashes)
- **Schema evolution** — Generate data matching a new schema before real data exists

### LLM-Generated Data Risks

- **Model collapse** — Training a model on synthetic data from another model can degrade quality over generations
- **Bias amplification** — LLMs may amplify biases present in their training data
- **Hallucinated facts** — Generated data may contain plausible but factually incorrect information
- **Lack of diversity** — LLMs may generate repetitive or stereotypical examples without careful prompting
- **Contamination** — Synthetic data may inadvertently overlap with evaluation benchmarks

### Data Lineage

Track the full provenance of synthetic datasets:

- **Source data** — What real data (if any) was used to train the generator
- **Generation method** — Which model, parameters, and random seeds were used
- **Validation results** — Fidelity, utility, and privacy scores
- **Downstream usage** — Which models or systems were trained on this synthetic data

## Benefits

1. **Privacy Preservation** — Share and use data without exposing real individuals
2. **Unlimited Scale** — Generate as much data as needed, no collection constraints
3. **Rare Event Coverage** — Create examples of edge cases and rare events for training
4. **Cost Reduction** — Cheaper than manual data collection and labeling
5. **Regulatory Compliance** — Use synthetic data where real data is restricted (GDPR, HIPAA)
6. **Faster Development** — Generate test data instantly without waiting for real data pipelines

## Trade-offs

| Advantage                            | Consideration                                              |
| ------------------------------------ | ---------------------------------------------------------- |
| Privacy-preserving data sharing      | Fidelity vs. privacy tension requires careful balancing    |
| Unlimited data volume                | Generator quality determines data usefulness               |
| Covers rare events and edge cases    | Generated edge cases may not reflect real-world patterns   |
| Faster than manual data collection   | Validation overhead to ensure quality and privacy          |
| Enables data sharing across teams    | Downstream model performance may differ from real data     |
| No labeling costs for generated data | LLM-generated data risks (bias, repetition, hallucination) |

## When to Use

✅ **Good fit for:**

- Training ML models when real data is scarce, expensive, or restricted by privacy regulations
- Generating evaluation and test datasets with known properties
- Augmenting underrepresented classes or rare events in imbalanced datasets
- Creating instruction-tuning data for fine-tuning language models
- Sharing realistic data across teams or organizations without privacy risk
- Testing data pipelines, ETL processes, and application prototypes

❌ **Not ideal for:**

- Domains where synthetic data fidelity cannot be validated against real data
- High-stakes decisions where model training exclusively on synthetic data is insufficient
- Cases where the data distribution is unknown or poorly understood
- When real data is abundant, cheap, and unrestricted — adding synthetic data may not help

## References

- [The Synthetic Data Vault (SDV) — Patki et al. (2016)](https://dai.lids.mit.edu/wp-content/uploads/2018/03/SDV.pdf)
- [CTGAN: Modeling Tabular Data using Conditional GAN — Xu et al. (2019)](https://arxiv.org/abs/1907.00503)
- [Self-Instruct: Aligning LMs with Self-Generated Instructions — Wang et al. (2023)](https://arxiv.org/abs/2212.10560)
- [Textbooks Are All You Need — Gunasekar et al. (2023)](https://arxiv.org/abs/2306.11644)
- [Synthetic Data — Microsoft Research](https://www.microsoft.com/en-us/research/project/synthetic-data/)
- [Gretel.ai — Synthetic Data Platform](https://gretel.ai/)
- [Mostly AI — Synthetic Data Generation](https://mostly.ai/)
- [Privacy in Synthetic Data Generation — Stadler et al. (2022)](https://arxiv.org/abs/2011.03640)
