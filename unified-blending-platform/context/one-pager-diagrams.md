# UBP Abstraction Layer — Visual Diagrams

> Companion to the 1-pager proposal. All diagrams are Mermaid format for embedding in Google Docs,
> Notion, or GitHub.

---

## 1. Class Diagram: Interfaces, Adapters, and Steps

The "aha" moment: any carousel type adapts to `FeedRow`, any ranking algorithm implements
`FeedRowRankingStep`, and the engine orchestrates them uniformly. Teams implement their own
adapters and steps — the engine doesn't change.

```mermaid
classDiagram
    direction TB

    class FeedRow {
        <<interface>>
        +id: String
        +type: RowType
        +score: Double
        +metadata: Map~String, Any~
        +applyBackTo()
    }

    class RowType {
        <<enum>>
        STORE_CAROUSEL
        ITEM_CAROUSEL
        DEAL_CAROUSEL
        STORE_COLLECTION
        COLLECTION_V2
        ITEM_COLLECTION
        MAP_CAROUSEL
        REELS_CAROUSEL
        STORE_ENTITY
    }

    class StoreCarouselRow {
        -carousel: StoreCarousel
        +applyBackTo()
    }
    class ItemCarouselRow {
        -carousel: ItemCarousel
        +applyBackTo()
    }
    class DealCarouselRow {
        -carousel: DealCarousel
        +applyBackTo()
    }
    class StoreCollectionRow {
        -collection: StoreCollection
        +applyBackTo()
    }
    class CollectionV2Row {
        -collection: CollectionV2
        +applyBackTo()
    }
    class ItemCollectionRow {
        -collection: ItemCollection
        +applyBackTo()
    }
    class MapCarouselRow {
        -carousel: MapCarousel
        +applyBackTo()
    }
    class ReelsCarouselRow {
        -carousel: ReelsCarousel
        +applyBackTo()
    }
    class StoreEntityRow {
        -entity: StoreEntity
        +applyBackTo()
    }

    FeedRow <|.. StoreCarouselRow
    FeedRow <|.. ItemCarouselRow
    FeedRow <|.. DealCarouselRow
    FeedRow <|.. StoreCollectionRow
    FeedRow <|.. CollectionV2Row
    FeedRow <|.. ItemCollectionRow
    FeedRow <|.. MapCarouselRow
    FeedRow <|.. ReelsCarouselRow
    FeedRow <|.. StoreEntityRow
    FeedRow --> RowType

    class FeedRowRankingStep {
        <<interface>>
        +stepType: String
        +paramsClass: Class~StepParams~
        +process(rows, context, params)
    }

    class StepParams {
        <<sealed interface>>
    }

    class ModelScoringStep {
        -entityScorer: EntityScorer
        +process(rows, context, params)
    }
    class MultiplierBoostStep {
        +process(rows, context, params)
    }
    class DiversityRerankStep {
        +process(rows, context, params)
    }
    class FixedPinningStep {
        +process(rows, context, params)
    }

    FeedRowRankingStep <|.. ModelScoringStep
    FeedRowRankingStep <|.. MultiplierBoostStep
    FeedRowRankingStep <|.. DiversityRerankStep
    FeedRowRankingStep <|.. FixedPinningStep
    FeedRowRankingStep --> StepParams

    class FeedRowRanker {
        -stepRegistry: Map~String, FeedRowRankingStep~
        -traceEmitter: UbpTraceEmitter
        +rank(rows, pipeline, context): List~FeedRow~
    }

    FeedRowRanker --> FeedRowRankingStep : dispatches to
    FeedRowRanker --> FeedRow : operates on
```

---

## 2. Sequence Diagram: Engine Dispatch Flow

Shows the full lifecycle of a ranking request through the UBP engine: adapt → score → boost →
diversify → pin → apply back. Params are injected at each step from config — steps never read
DVs internally.

```mermaid
sequenceDiagram
    participant PP as PostProcessor
    participant Adapter as FeedRow Adapters
    participant Engine as FeedRowRanker
    participant Registry as StepRegistry
    participant Score as ModelScoringStep
    participant Boost as MultiplierBoostStep
    participant Diversity as DiversityRerankStep
    participant Pin as FixedPinningStep
    participant Trace as TraceEmitter

    PP->>Adapter: toFeedRow() for each carousel
    Note over Adapter: 9 types → uniform FeedRow list

    PP->>Engine: rank(rows, pipeline, context)

    loop For each step in pipeline config
        Engine->>Registry: lookup(stepConfig.type)
        Registry-->>Engine: step instance
        Engine->>Engine: deserialize params → StepParams

        alt step = MODEL_SCORING
            Engine->>Score: process(rows, context, params)
            Score->>Score: entityScorer.score() → Sibyl gRPC
            Score-->>Engine: rows with scores set
        else step = MULTIPLIER_BOOST
            Engine->>Boost: process(rows, context, params)
            Boost->>Boost: BlendingUtil.blendBundle()
            Boost-->>Engine: rows with boosted scores
        else step = DIVERSITY_RERANK
            Engine->>Diversity: process(rows, context, params)
            Diversity->>Diversity: BlendingUtil.rerankEntitiesWithDiversity()
            Diversity-->>Engine: rows reranked
        else step = FIXED_PINNING
            Engine->>Pin: process(rows, context, params)
            Pin->>Pin: BoostingBundle.boosted() + RankingBundle.ranked()
            Pin-->>Engine: rows with pins applied
        end

        opt emit_trace = true
            Engine->>Trace: recordStep(row_id, step_id, score_before, score_after)
        end
    end

    Engine-->>PP: rows sorted by final score

    PP->>Adapter: applyBackTo() for each FeedRow
    Note over Adapter: Writes scores back to original domain objects
```

---

## 3. Before / After: Current Spaghetti vs Clean Pipeline

### Before: Inline calls through utility objects

No interfaces, no boundaries. Understanding one stage requires reading all stages.

```mermaid
flowchart TB
    subgraph current["Current: Inline Method Calls (No Interfaces)"]
        direction TB
        A["reOrderGlobalEntitiesV2()"] --> B["rankAndDedupeContent()"]
        B --> C["rankAndMergeContent()"]
        C --> D["rankContent()"]
        D --> E["BaseEntityRankerConfiguration.rank()"]

        E --> F["getEntities()
        Flatten 9+ types via type-checks"]
        E --> G["getScoreBundle()
        → EntityScorer → SibylRegressor
        → gRPC to Sibyl"]
        E --> H["getBoostBundle()
        → BoostingBundle.boosted()
        Reads DVs + runtime JSON internally"]
        E --> I["getRankingBundle()
        → RankingBundle.ranked()
        Pin/flow separation"]
        E --> J["getRankableContent()
        Re-assemble typed containers"]

        G -.->|"reads config from"| K["3 runtime JSONs
        5+ DV keys
        Hardcoded constants"]
        H -.->|"reads config from"| K
    end

    style current fill:#fff3f3,stroke:#cc0000
    style K fill:#ffcccc,stroke:#cc0000
```

### After: Named steps with typed params

Each step is independent, testable, and configurable. Params flow in from config — no internal
DV reads.

```mermaid
flowchart TB
    subgraph proposed["Proposed: Config-Driven Pipeline"]
        direction TB
        A2["reOrderGlobalEntitiesV2()"] --> B2{"DV: ubpRolloutEnabled?"}
        B2 -->|Yes| C2["Adapt → FeedRow list"]
        B2 -->|No| OLD["Old path (unchanged)"]

        C2 --> D2["FeedRowRanker.rank()"]

        D2 --> E2["ModelScoringStep
        calls EntityScorer.score()
        Same Sibyl gRPC"]
        E2 --> F2["MultiplierBoostStep
        calls BlendingUtil.blendBundle()
        Same in-memory math"]
        F2 --> G2["DiversityRerankStep
        calls BlendingUtil.rerankEntitiesWithDiversity()
        Same in-memory math"]
        G2 --> H2["FixedPinningStep
        calls BoostingBundle.boosted()
        Same in-memory math"]

        H2 --> I2["applyBackTo()
        Write scores to original objects"]

        J2["Pipeline Config JSON
        (one source of truth)"] -.->|"params injected"| E2
        J2 -.->|"params injected"| F2
        J2 -.->|"params injected"| G2
        J2 -.->|"params injected"| H2
    end

    style proposed fill:#f3fff3,stroke:#00aa00
    style J2 fill:#ccffcc,stroke:#00aa00
    style OLD fill:#ffffcc,stroke:#aaaa00
```

---

## 4. Strangler Fig: Shadow → Rollout Migration

Two phases of wiring. Shadow runs both paths in parallel (new path's result is discarded).
Rollout switches the primary path via DV.

```mermaid
flowchart TB
    subgraph shadow["Phase 1: Shadow Mode"]
        direction TB
        R1["Request"] --> PP1["PostProcessor"]
        PP1 --> OLD1["Old Path
        rankAndDedupeContent()"]
        OLD1 --> RESULT1["Result → User"]

        PP1 --> DV1{"DV: ubpShadowEnabled?"}
        DV1 -->|Yes| NEW1["UBP Engine
        FeedRowRanker.rank()"]
        DV1 -->|No| SKIP1["Skip"]
        NEW1 --> CMP["Compare sort orders
        Log divergences"]
        NEW1 --> CATCH["catch(Exception)
        Swallow — never propagates"]
        CMP --> DISCARD["Discard shadow result"]

        RESULT1 ~~~ CMP
    end

    subgraph rollout["Phase 2: Rollout Mode"]
        direction TB
        R2["Request"] --> PP2["PostProcessor"]
        PP2 --> DV2{"DV: ubpRolloutEnabled?"}
        DV2 -->|Yes| NEW2["UBP Engine
        FeedRowRanker.rank()"]
        DV2 -->|No| OLD2["Old Path
        rankAndDedupeContent()
        (byte-for-byte unchanged)"]
        NEW2 --> RESULT2a["Result → User"]
        OLD2 --> RESULT2b["Result → User"]
    end

    shadow -.->|"divergence_count = 0
    sustained"| rollout

    style shadow fill:#fff8e6,stroke:#cc8800
    style rollout fill:#e6fff0,stroke:#00aa44
    style DISCARD fill:#ffcccc,stroke:#cc0000
    style CATCH fill:#ffcccc,stroke:#cc0000
```

---

## 5. Horizontal Mirroring: Same Architecture, Different Types

Horizontal ranking follows the identical pattern. `RowItem` mirrors `FeedRow`; `RowItemRankingStep`
mirrors `FeedRowRankingStep`. The engine shape is the same — only the abstraction type changes.

```mermaid
flowchart LR
    subgraph vertical["Vertical Ranking (Phase 1)"]
        direction TB
        V_IF["FeedRow
        (interface)"]
        V_ADAPT["9 Adapters
        StoreCarouselRow
        ItemCarouselRow
        ..."]
        V_STEP["FeedRowRankingStep
        (interface)"]
        V_IMPL["ModelScoringStep
        MultiplierBoostStep
        DiversityRerankStep
        FixedPinningStep"]
        V_ENG["FeedRowRanker
        (engine)"]
        V_ENTRY["Incision:
        PostProcessor
        .reOrderGlobalEntitiesV2()"]

        V_ENTRY --> V_ENG
        V_ENG --> V_IF
        V_ENG --> V_STEP
        V_IF --- V_ADAPT
        V_STEP --- V_IMPL
    end

    subgraph horizontal["Horizontal Ranking (follows vertical)"]
        direction TB
        H_IF["RowItem
        (interface)"]
        H_ADAPT["Adapters
        StoreRowItem
        ItemRowItem
        ..."]
        H_STEP["RowItemRankingStep
        (interface)"]
        H_IMPL["ModelScoringStep
        ScoreModifierStep
        BusinessRulesSortStep
        OrderHistoryRerankStep"]
        H_ENG["RowItemRanker
        (engine)"]
        H_ENTRY["Incision:
        StoreRanker
        .modifyLiteStoreCollection()"]

        H_ENTRY --> H_ENG
        H_ENG --> H_IF
        H_ENG --> H_STEP
        H_IF --- H_ADAPT
        H_STEP --- H_IMPL
    end

    vertical -.->|"Same architecture
    Proven first, then mirrored"| horizontal

    style vertical fill:#e6f0ff,stroke:#0055cc
    style horizontal fill:#f0e6ff,stroke:#7700cc
```

---

## 6. Carousel Onboarding: Before vs After

The "aha" for product teams: adding a new carousel type goes from touching 10+ files to writing
1 adapter class.

```mermaid
flowchart TB
    subgraph before["Before: New Carousel Type Onboarding"]
        direction TB
        NEW_TYPE["New carousel type
        (e.g., ReelsCarousel)"] --> F1["Modify BaseEntityRankerConfiguration
        Add type-check branch"]
        NEW_TYPE --> F2["Modify EntityScorer
        Add feature extraction case"]
        NEW_TYPE --> F3["Modify BlendingUtil
        Add blending case"]
        NEW_TYPE --> F4["Modify BoostingBundle
        Add boosting case"]
        NEW_TYPE --> F5["Modify RankingBundle
        Add ranking case"]
        NEW_TYPE --> F6["Modify NonRankableHomepageOrderingUtil
        Add fixup case"]
        NEW_TYPE --> F7["Modify PostProcessor
        Add serialization case"]
        NEW_TYPE --> F8["Modify PinnedCarouselUtil
        Add pinning case"]
        NEW_TYPE --> F9["Modify ScorableEntity
        Add new subtype"]
        NEW_TYPE --> F10["Modify RankableContent
        Add new container field"]

        F1 ~~~ NOTE1["10+ files touched
        Deep HP knowledge required
        Core team pairing needed
        2-3 weeks"]
    end

    subgraph after["After: New Carousel Type Onboarding"]
        direction TB
        NEW_TYPE2["New carousel type
        (e.g., ReelsCarousel)"] --> A1["Write ReelsCarouselRow
        implements FeedRow
        (1 adapter class)"]

        A1 --> A2["toFeedRow(): wrap domain object
        applyBackTo(): write score back"]

        A2 --> DONE["Done.
        Engine and all steps work automatically.
        No other files touched."]
    end

    style before fill:#fff3f3,stroke:#cc0000
    style after fill:#f3fff3,stroke:#00aa00
    style NOTE1 fill:#ffcccc,stroke:#cc0000
    style DONE fill:#ccffcc,stroke:#00aa00
```
