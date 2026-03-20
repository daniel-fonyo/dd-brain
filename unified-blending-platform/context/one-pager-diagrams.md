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

    class StepType {
        <<string constants>>
        MODEL_SCORING
        BOOST_AND_RANK
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
    class BoostAndRankParams {
        +boostByPositionEnabled: Boolean
        +dealCarouselScoreMultiplier: Double
        +boostByPositionAllowList: List~String~
    }

    StepParams <|.. ModelScoringParams
    StepParams <|.. BoostAndRankParams

    class ModelScoringStep {
        -entityScorer: EntityScorer
        +process(rows, context, params)
    }
    class BoostAndRankStep {
        +process(rows, context, params)
    }

    FeedRowRankingStep <|.. ModelScoringStep
    FeedRowRankingStep <|.. BoostAndRankStep
    FeedRowRankingStep --> StepType : type
    FeedRowRankingStep --> StepParams : params

    ModelScoringStep --> ModelScoringParams
    BoostAndRankStep --> BoostAndRankParams

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

Side-by-side contrast. Left: the current code branches on carousel type at every ranking stage —
9 types x 4 stages = type-checks everywhere. Right: adapt once, rank uniformly.

```mermaid
flowchart TB
    subgraph before["Before: Type-Checks at Every Stage"]
        direction TB
        S1["Score Stage"] --> B1{"if StoreCarousel?
        if ItemCarousel?
        if DealCarousel?
        ... 9 branches"}
        S2["Blend Stage"] --> B2{"if StoreCarousel?
        if ItemCarousel?
        if DealCarousel?
        ... 9 branches"}
        S3["Boost Stage"] --> B3{"if StoreCarousel?
        if ItemCarousel?
        if DealCarousel?
        ... 9 branches"}
        S4["Pin Stage"] --> B4{"if StoreCarousel?
        if ItemCarousel?
        if DealCarousel?
        ... 9 branches"}

        B1 --> R1["9 type-specific
        scoring paths"]
        B2 --> R2["9 type-specific
        blending paths"]
        B3 --> R3["9 type-specific
        boosting paths"]
        B4 --> R4["9 type-specific
        pinning paths"]

        NOTE["36 type-check branches
        across 4 stages"]
    end

    subgraph after["After: Adapt Once, Rank Uniformly"]
        direction TB
        ADAPT["Adapt once
        toFeedRow()"] --> UNI["Uniform FeedRow list"]
        UNI --> P1["MODEL_SCORING"]
        P1 --> P2["BOOST_AND_RANK"]
        P2 --> BACK["applyBackTo()
        Write back once"]

        NOTE2["0 type-checks in pipeline
        Each step sees only FeedRow"]
    end

    style before fill:#fff3f3,stroke:#cc0000
    style after fill:#f3fff3,stroke:#00aa00
    style NOTE fill:#ffcccc,stroke:#cc0000
    style NOTE2 fill:#ccffcc,stroke:#00aa00
```

---

## 2B. The Funnel: Adapt Once, Rank Uniformly, Apply Back

The "aha" visual. 9 diverse carousel types fan in through adapters → converge to a uniform
FeedRow list → pass through the sequential step pipeline → fan out via applyBackTo() back to
original domain objects.

```mermaid
flowchart LR
    subgraph sources["9 Carousel Types"]
        direction TB
        T1["StoreCarousel"]
        T2["ItemCarousel"]
        T3["DealCarousel"]
        T4["StoreCollection"]
        T5["CollectionV2"]
        T6["ItemCollection"]
        T7["MapCarousel"]
        T8["ReelsCarousel"]
        T9["StoreEntity"]
    end

    subgraph adapt["Adapt"]
        direction TB
        A["toFeedRow()
        1 adapter per type"]
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

    subgraph pipeline["Uniform FeedRow Pipeline"]
        direction TB
        STEP1["MODEL_SCORING
        ModelScoringParams"]
        STEP2["BOOST_AND_RANK
        BoostAndRankParams"]
        STEP1 --> STEP2
    end

    A --> STEP1

    subgraph writeback["Apply Back"]
        direction TB
        WB["applyBackTo()
        Scores → original objects"]
    end

    STEP2 --> WB

    subgraph outputs["Original Domain Objects"]
        direction TB
        O1["StoreCarousel"]
        O2["ItemCarousel"]
        O3["DealCarousel"]
        O4["StoreCollection"]
        O5["CollectionV2"]
        O6["ItemCollection"]
        O7["MapCarousel"]
        O8["ReelsCarousel"]
        O9["StoreEntity"]
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

    style sources fill:#fff3f3,stroke:#cc0000
    style adapt fill:#ffffcc,stroke:#aaaa00
    style pipeline fill:#e6f0ff,stroke:#0055cc
    style writeback fill:#ffffcc,stroke:#aaaa00
    style outputs fill:#f3fff3,stroke:#00aa00
```

---

## 3. Strangler Fig: Shadow → Rollout Migration

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

## 4A. Horizontal Mirroring: Same Architecture, Different Types

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
        BoostAndRankStep"]
        V_TYPES["Step Types:
        MODEL_SCORING
        BOOST_AND_RANK"]
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
        V_IMPL --- V_TYPES
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
        RankingSortStep"]
        H_TYPES["Step Types:
        MODEL_SCORING
        RANKING_SORT"]
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
        H_IMPL --- H_TYPES
    end

    vertical -.->|"Same architecture
    Proven first, then mirrored"| horizontal

    style vertical fill:#e6f0ff,stroke:#0055cc
    style horizontal fill:#f0e6ff,stroke:#7700cc
```

---

## 4B. The Horizontal Funnel: Adapt Once, Rank Stores Uniformly

Same "aha" as 2B but for within-carousel ranking. Today, each RankingType (30+) has its own
bespoke sort logic scattered across StoreRanker, CampaignRanker, and StoreCarouselService.
With UBP: adapt each store to RowItem once → uniform pipeline → applyBackTo().

```mermaid
flowchart LR
    subgraph sources["Store Sources (by RankingType)"]
        direction TB
        T1["StoreEntity
        (standard carousel)"]
        T2["StoreEntity
        (collection)"]
        T3["DealStore
        (campaign/deals)"]
        T4["ItemStoreEntity
        (item carousel)"]
        T5["StoreEntity
        (save list)"]
        T6["StoreEntity
        (search results)"]
    end

    subgraph adapt["Adapt"]
        direction TB
        A["toRowItem()
        1 adapter per source"]
    end

    T1 --> A
    T2 --> A
    T3 --> A
    T4 --> A
    T5 --> A
    T6 --> A

    subgraph pipeline["Uniform RowItem Pipeline"]
        direction TB
        STEP1["MODEL_SCORING
        ModelScoringParams"]
        STEP2["RANKING_SORT
        RankingSortParams"]
        STEP1 --> STEP2
    end

    A --> STEP1

    subgraph writeback["Apply Back"]
        direction TB
        WB["applyBackTo()
        Scores → original store objects"]
    end

    STEP2 --> WB

    subgraph outputs["Original Store Objects"]
        direction TB
        O1["StoreEntity"]
        O2["StoreEntity"]
        O3["DealStore"]
        O4["ItemStoreEntity"]
        O5["StoreEntity"]
        O6["StoreEntity"]
    end

    WB --> O1
    WB --> O2
    WB --> O3
    WB --> O4
    WB --> O5
    WB --> O6

    style sources fill:#fff3f3,stroke:#cc0000
    style adapt fill:#ffffcc,stroke:#aaaa00
    style pipeline fill:#f0e6ff,stroke:#7700cc
    style writeback fill:#ffffcc,stroke:#aaaa00
    style outputs fill:#f3fff3,stroke:#00aa00
```

---

## 5. Carousel Onboarding: Before vs After

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
