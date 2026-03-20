# UBP Abstraction Layer — Visual Diagrams

> Companion to the 1-pager proposal. All diagrams are Mermaid format for embedding in Google Docs,
> Notion, or GitHub.

---

## 1. Class Diagram: Interfaces, Adapters, and Steps

<!-- Diagram: Full class hierarchy
     Reason: Show all interfaces, adapters, steps, and params and how they connect
     Aha: Any carousel type adapts to FeedRow, any ranking algorithm implements FeedRowRankingStep — the engine orchestrates uniformly -->

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

    class StepType {
        <<string constants>>
        MODEL_SCORING
        MULTIPLIER_BOOST
        DIVERSITY_RERANK
        POSITION_BOOSTING
        FIXED_PINNING
    }

    class FeedRowRankingStep {
        <<interface>>
        +stepType: String
        +paramsClass: Class~StepParams~
        +process(rows, context, params)
    }

    class StepParams {
        <<sealed interface>>
    }

    class ModelScoringParams {
        +predictorRef: String?
        +predictorName: String?
        +modelName: String?
    }
    class MultiplierBoostParams {
        +calibrationConfig: CalibrationConfig
        +intentScoringConfig: IntentScoringConfig
        +verticalBoostWeights: VerticalBoostWeights
    }
    class DiversityRerankParams {
        +enabled: Boolean
        +diversityScoringParams: DiversityScoringParams
    }
    class PositionBoostingParams {
        +dealCarouselMultiplier: Double
        +boostByPositionAllowList: List~String~
        +nvUnpinEnabled: Boolean
    }
    class FixedPinningParams {
        +rules: List~PinRule~
    }
    class PinRule {
        +rowId: String
        +position: Int
    }

    StepParams <|.. ModelScoringParams
    StepParams <|.. MultiplierBoostParams
    StepParams <|.. DiversityRerankParams
    StepParams <|.. PositionBoostingParams
    StepParams <|.. FixedPinningParams
    FixedPinningParams --> PinRule

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
    class PositionBoostingStep {
        +process(rows, context, params)
    }
    class FixedPinningStep {
        +process(rows, context, params)
    }

    FeedRowRankingStep <|.. ModelScoringStep
    FeedRowRankingStep <|.. MultiplierBoostStep
    FeedRowRankingStep <|.. DiversityRerankStep
    FeedRowRankingStep <|.. PositionBoostingStep
    FeedRowRankingStep <|.. FixedPinningStep
    FeedRowRankingStep --> StepType : type
    FeedRowRankingStep --> StepParams : params

    ModelScoringStep --> ModelScoringParams
    MultiplierBoostStep --> MultiplierBoostParams
    DiversityRerankStep --> DiversityRerankParams
    PositionBoostingStep --> PositionBoostingParams
    FixedPinningStep --> FixedPinningParams

    class FeedRowRanker {
        -stepRegistry: Map~String, FeedRowRankingStep~
        -traceEmitter: UbpTraceEmitter
        +rank(rows, pipeline, context): List~FeedRow~
    }

    FeedRowRanker --> FeedRowRankingStep : dispatches to
    FeedRowRanker --> FeedRow : operates on
```

---

## 2A. Before vs After: Type-Check Branching at Every Stage

<!-- Diagram: Before vs After contrast
     Reason: Show the scale of the problem — 36 type-check branches — next to the clean alternative
     Aha: "Before" is 9x4=36 branches; "After" is 0 type-checks in the pipeline -->

```mermaid
flowchart TB
    subgraph before["  Before: Type-Checks at Every Stage  "]
        direction TB
        S1("Score Stage"):::domain --> B1{"if StoreCarousel?
        if ItemCarousel?
        if DealCarousel?
        ... 9 branches"}:::domain
        S2("Blend Stage"):::domain --> B2{"if StoreCarousel?
        if ItemCarousel?
        if DealCarousel?
        ... 9 branches"}:::domain
        S3("Boost Stage"):::domain --> B3{"if StoreCarousel?
        if ItemCarousel?
        if DealCarousel?
        ... 9 branches"}:::domain
        S4("Pin Stage"):::domain --> B4{"if StoreCarousel?
        if ItemCarousel?
        if DealCarousel?
        ... 9 branches"}:::domain

        B1 --> R1("9 type-specific
        scoring paths"):::domain
        B2 --> R2("9 type-specific
        blending paths"):::domain
        B3 --> R3("9 type-specific
        boosting paths"):::domain
        B4 --> R4("9 type-specific
        pinning paths"):::domain

        NOTE("36 type-check branches
        across 4 stages"):::callout_problem
    end

    subgraph after["  After: Adapt Once, Rank Uniformly  "]
        direction TB
        ADAPT("Adapt once
        toFeedRow()"):::hero --> UNI("Uniform FeedRow list"):::step
        UNI --> P1("MODEL_SCORING"):::step
        P1 --> P2("MULTIPLIER_BOOST"):::step
        P2 --> P3("DIVERSITY_RERANK"):::step
        P3 --> P4("POSITION_BOOSTING"):::step
        P4 --> P5("FIXED_PINNING"):::step
        P5 --> BACK("applyBackTo()
        Write back once"):::hero

        NOTE2("0 type-checks in pipeline
        Each step sees only FeedRow"):::callout_good
    end

    %% Gray = the existing mess. Red = the action. Blue = the new clean flow.
    classDef domain fill:#EAEAED,stroke:#B8B8C2,color:#505058,stroke-width:1px
    classDef hero fill:#FF3008,stroke:#D42807,color:#FFFFFF,stroke-width:1.5px
    classDef step fill:#DCEEFB,stroke:#7BBCE0,color:#1A3A50,stroke-width:1px
    classDef callout_problem fill:#FFF0EB,stroke:#FF3008,color:#FF3008,stroke-width:1px
    classDef callout_good fill:#DCEEFB,stroke:#7BBCE0,color:#1A3A50,stroke-width:1px

    style before fill:transparent,stroke:#C8C8D0,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
    style after fill:transparent,stroke:#A0CCE8,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
```

---

## 2B. The Funnel: Adapt Once, Rank Uniformly, Apply Back

<!-- Diagram: Full funnel with all 9 types and params
     Reason: Show the complete fan-in → pipeline → fan-out flow with param details
     Aha: 9 diverse types converge through adapters into one uniform pipeline, then fan back out -->

```mermaid
flowchart LR
    subgraph sources["  9 Carousel Types  "]
        direction TB
        T1("StoreCarousel"):::domain
        T2("ItemCarousel"):::domain
        T3("DealCarousel"):::domain
        T4("StoreCollection"):::domain
        T5("CollectionV2"):::domain
        T6("ItemCollection"):::domain
        T7("MapCarousel"):::domain
        T8("ReelsCarousel"):::domain
        T9("StoreEntity"):::domain
    end

    subgraph adapt["  Adapt  "]
        direction TB
        A("toFeedRow()
        1 adapter per type"):::hero
    end

    T1 --> A
    T2 --> A
    T3 --> A
    T4 --> A
    T5 --> A
    T6 --> A
    T7 --> A
    T8 --> A
    T9 --> A

    subgraph pipeline["  Uniform FeedRow Pipeline  "]
        direction TB
        STEP1("MODEL_SCORING
        ModelScoringParams"):::step
        STEP2("MULTIPLIER_BOOST
        MultiplierBoostParams"):::step
        STEP3("DIVERSITY_RERANK
        DiversityRerankParams"):::step
        STEP4("POSITION_BOOSTING
        PositionBoostingParams"):::step
        STEP5("FIXED_PINNING
        FixedPinningParams"):::step
        STEP1 --> STEP2 --> STEP3 --> STEP4 --> STEP5
    end

    A --> STEP1

    subgraph writeback["  Apply Back  "]
        direction TB
        WB("applyBackTo()
        Scores → original objects"):::hero
    end

    STEP5 --> WB

    subgraph outputs["  Original Domain Objects  "]
        direction TB
        O1("StoreCarousel"):::domain
        O2("ItemCarousel"):::domain
        O3("DealCarousel"):::domain
        O4("StoreCollection"):::domain
        O5("CollectionV2"):::domain
        O6("ItemCollection"):::domain
        O7("MapCarousel"):::domain
        O8("ReelsCarousel"):::domain
        O9("StoreEntity"):::domain
    end

    WB --> O1
    WB --> O2
    WB --> O3
    WB --> O4
    WB --> O5
    WB --> O6
    WB --> O7
    WB --> O8
    WB --> O9

    classDef domain fill:#EAEAED,stroke:#B8B8C2,color:#505058,stroke-width:1px
    classDef hero fill:#FF3008,stroke:#D42807,color:#FFFFFF,stroke-width:1.5px
    classDef step fill:#DCEEFB,stroke:#7BBCE0,color:#1A3A50,stroke-width:1px

    style sources fill:transparent,stroke:#C8C8D0,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
    style adapt fill:transparent,stroke:#E0A090,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
    style pipeline fill:transparent,stroke:#A0CCE8,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
    style writeback fill:transparent,stroke:#E0A090,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
    style outputs fill:transparent,stroke:#C8C8D0,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
```

---

## 3. Strangler Fig: Shadow → Rollout Migration

<!-- Diagram: Two-phase migration with safety mechanisms
     Reason: Show every safety mechanism — catch, discard, DV gates — and how the old path is never at risk
     Aha: The old path is NEVER removed. Shadow can never affect users. Rollout is a DV flip. -->

```mermaid
flowchart TB
    subgraph shadow["  Phase 1: Shadow Mode  "]
        direction TB
        R1("Request"):::domain --> PP1("PostProcessor"):::domain
        PP1 --> OLD1("Old Path
        rankAndDedupeContent()"):::domain
        OLD1 --> RESULT1("Result → User"):::domain

        PP1 --> DV1{"DV: ubpShadowEnabled?"}:::domain
        DV1 -->|Yes| NEW1("UBP Engine
        FeedRowRanker.rank()"):::step
        DV1 -->|No| SKIP1("Skip"):::domain
        NEW1 --> CMP("Compare sort orders
        Log divergences"):::step
        NEW1 --> CATCH("catch Exception
        Swallow — never propagates"):::discard
        CMP --> DISCARD("Discard shadow result"):::discard

        RESULT1 ~~~ CMP
    end

    subgraph rollout["  Phase 2: Rollout Mode  "]
        direction TB
        R2("Request"):::domain --> PP2("PostProcessor"):::domain
        PP2 --> DV2{"ubpRolloutEnabled?"}:::domain
        DV2 -->|Yes| NEW2("UBP Engine"):::hero
        DV2 -->|No| OLD2("Old Path
        unchanged"):::domain
        NEW2 --> RESULT2a("Result → User"):::domain
        OLD2 --> RESULT2b("Result → User"):::domain
    end

    shadow -.->|"divergence_count = 0
    sustained"| rollout

    %% Gray = existing/plumbing. Blue = new path (shadow, experimental). Red = new path (live). Red tint = discard/danger.
    classDef domain fill:#EAEAED,stroke:#B8B8C2,color:#505058,stroke-width:1px
    classDef step fill:#DCEEFB,stroke:#7BBCE0,color:#1A3A50,stroke-width:1px
    classDef hero fill:#FF3008,stroke:#D42807,color:#FFFFFF,stroke-width:1.5px
    classDef discard fill:#FFF0EB,stroke:#FF3008,color:#FF3008,stroke-width:1px

    style shadow fill:transparent,stroke:#A0CCE8,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
    style rollout fill:transparent,stroke:#E0A090,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
```

---

## 4A. Horizontal Mirroring: Same Architecture, Different Types

<!-- Diagram: Vertical and horizontal side by side with full detail
     Reason: Show that horizontal is the exact same architecture — only types and step names differ
     Aha: Once you understand vertical, horizontal is a copy-paste with different names -->

```mermaid
flowchart LR
    subgraph vertical["  Vertical Ranking (Phase 1)  "]
        direction TB
        V_IF("FeedRow
        interface"):::step
        V_ADAPT("9 Adapters
        StoreCarouselRow
        ItemCarouselRow
        ..."):::domain
        V_STEP("FeedRowRankingStep
        interface"):::step
        V_IMPL("ModelScoringStep
        MultiplierBoostStep
        DiversityRerankStep
        PositionBoostingStep
        FixedPinningStep"):::domain
        V_TYPES("Step Types:
        MODEL_SCORING
        MULTIPLIER_BOOST
        DIVERSITY_RERANK
        POSITION_BOOSTING
        FIXED_PINNING"):::domain
        V_ENG("FeedRowRanker
        engine"):::step
        V_ENTRY("Incision:
        PostProcessor
        .reOrderGlobalEntitiesV2()"):::domain

        V_ENTRY --> V_ENG
        V_ENG --> V_IF
        V_ENG --> V_STEP
        V_IF --- V_ADAPT
        V_STEP --- V_IMPL
        V_IMPL --- V_TYPES
    end

    subgraph horizontal["  Horizontal Ranking (follows vertical)  "]
        direction TB
        H_IF("RowItem
        interface"):::step
        H_ADAPT("Adapters
        StoreRowItem
        ItemRowItem
        ..."):::domain
        H_STEP("RowItemRankingStep
        interface"):::step
        H_IMPL("ModelScoringStep
        RankingSortStep"):::domain
        H_TYPES("Step Types:
        MODEL_SCORING
        RANKING_SORT"):::domain
        H_ENG("RowItemRanker
        engine"):::step
        H_ENTRY("Incision:
        StoreRanker
        .modifyLiteStoreCollection()"):::domain

        H_ENTRY --> H_ENG
        H_ENG --> H_IF
        H_ENG --> H_STEP
        H_IF --- H_ADAPT
        H_STEP --- H_IMPL
        H_IMPL --- H_TYPES
    end

    vertical -.->|"Same architecture
    Proven first, then mirrored"| horizontal

    %% Blue = new interfaces/engines. Gray = existing code, implementations, step lists.
    classDef domain fill:#EAEAED,stroke:#B8B8C2,color:#505058,stroke-width:1px
    classDef step fill:#DCEEFB,stroke:#7BBCE0,color:#1A3A50,stroke-width:1px

    style vertical fill:transparent,stroke:#A0CCE8,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
    style horizontal fill:transparent,stroke:#A0CCE8,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
```

---

## 4B. The Horizontal Funnel: Adapt Once, Rank Stores Uniformly

<!-- Diagram: Horizontal funnel — store types to RowItem pipeline
     Reason: Show that within-carousel ranking follows the exact same adapt → rank → apply pattern
     Aha: 30+ RankingTypes with scattered sort logic collapse into one uniform RowItem pipeline -->

```mermaid
flowchart LR
    subgraph sources["  Store Sources (by RankingType)  "]
        direction TB
        T1("StoreEntity
        standard carousel"):::domain
        T2("StoreEntity
        collection"):::domain
        T3("DealStore
        campaign/deals"):::domain
        T4("ItemStoreEntity
        item carousel"):::domain
        T5("StoreEntity
        save list"):::domain
        T6("StoreEntity
        search results"):::domain
    end

    subgraph adapt["  Adapt  "]
        direction TB
        A("toRowItem()
        1 adapter per source"):::hero
    end

    T1 --> A
    T2 --> A
    T3 --> A
    T4 --> A
    T5 --> A
    T6 --> A

    subgraph pipeline["  Uniform RowItem Pipeline  "]
        direction TB
        STEP1("MODEL_SCORING
        ModelScoringParams"):::step
        STEP2("RANKING_SORT
        RankingSortParams"):::step
        STEP1 --> STEP2
    end

    A --> STEP1

    subgraph writeback["  Apply Back  "]
        direction TB
        WB("applyBackTo()
        Scores → original store objects"):::hero
    end

    STEP2 --> WB

    subgraph outputs["  Original Store Objects  "]
        direction TB
        O1("StoreEntity"):::domain
        O2("StoreEntity"):::domain
        O3("DealStore"):::domain
        O4("ItemStoreEntity"):::domain
        O5("StoreEntity"):::domain
        O6("StoreEntity"):::domain
    end

    WB --> O1
    WB --> O2
    WB --> O3
    WB --> O4
    WB --> O5
    WB --> O6

    classDef domain fill:#EAEAED,stroke:#B8B8C2,color:#505058,stroke-width:1px
    classDef hero fill:#FF3008,stroke:#D42807,color:#FFFFFF,stroke-width:1.5px
    classDef step fill:#DCEEFB,stroke:#7BBCE0,color:#1A3A50,stroke-width:1px

    style sources fill:transparent,stroke:#C8C8D0,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
    style adapt fill:transparent,stroke:#E0A090,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
    style pipeline fill:transparent,stroke:#A0CCE8,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
    style writeback fill:transparent,stroke:#E0A090,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
    style outputs fill:transparent,stroke:#C8C8D0,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
```

---

## 5. Carousel Onboarding: Before vs After

<!-- Diagram: New carousel type onboarding comparison
     Reason: Show the cost reduction — 10+ files to 1 adapter class
     Aha: Adding a new carousel type goes from 2-3 weeks of deep HP knowledge to writing 1 class -->

```mermaid
flowchart TB
    subgraph before["  Before: New Carousel Type Onboarding  "]
        direction TB
        NEW_TYPE("New carousel type
        e.g., ReelsCarousel"):::domain --> F1("Modify BaseEntityRankerConfiguration
        Add type-check branch"):::domain
        NEW_TYPE --> F2("Modify EntityScorer
        Add feature extraction case"):::domain
        NEW_TYPE --> F3("Modify BlendingUtil
        Add blending case"):::domain
        NEW_TYPE --> F4("Modify BoostingBundle
        Add boosting case"):::domain
        NEW_TYPE --> F5("Modify RankingBundle
        Add ranking case"):::domain
        NEW_TYPE --> F6("Modify NonRankableHomepageOrderingUtil
        Add fixup case"):::domain
        NEW_TYPE --> F7("Modify PostProcessor
        Add serialization case"):::domain
        NEW_TYPE --> F8("Modify PinnedCarouselUtil
        Add pinning case"):::domain
        NEW_TYPE --> F9("Modify ScorableEntity
        Add new subtype"):::domain
        NEW_TYPE --> F10("Modify RankableContent
        Add new container field"):::domain

        F1 ~~~ NOTE1("10+ files touched
        Deep HP knowledge required
        Core team pairing needed
        2-3 weeks"):::callout_problem
    end

    subgraph after["  After: New Carousel Type Onboarding  "]
        direction TB
        NEW_TYPE2("New carousel type
        e.g., ReelsCarousel"):::domain --> A1("Write ReelsCarouselRow
        implements FeedRow
        1 adapter class"):::hero

        A1 --> A2("toFeedRow(): wrap domain object
        applyBackTo(): write score back"):::step

        A2 --> DONE("Done.
        Engine and all steps work automatically.
        No other files touched."):::callout_good
    end

    %% Gray = existing code/mess. Red = the action. Blue = the new clean path. Red tint = the problem callout.
    classDef domain fill:#EAEAED,stroke:#B8B8C2,color:#505058,stroke-width:1px
    classDef hero fill:#FF3008,stroke:#D42807,color:#FFFFFF,stroke-width:1.5px
    classDef step fill:#DCEEFB,stroke:#7BBCE0,color:#1A3A50,stroke-width:1px
    classDef callout_problem fill:#FFF0EB,stroke:#FF3008,color:#FF3008,stroke-width:1px
    classDef callout_good fill:#DCEEFB,stroke:#7BBCE0,color:#1A3A50,stroke-width:1px

    style before fill:transparent,stroke:#C8C8D0,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
    style after fill:transparent,stroke:#A0CCE8,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
```
