# RFC Mermaid Diagrams — Diagrams 4–10

> Brand-styled mermaid diagrams for the Ranking Abstraction Layer RFC.
> Follows `rfc-guide/CLAUDE.md` design guidelines and DoorDash Spark palette.
> All diagrams reflect the **shipped Phase 1 implementation**.

**Color legend (consistent across all diagrams):**

| Color | Role | ClassDef |
|---|---|---|
| Red (`#FF3008`) | The action / transformation this RFC proposes | `hero` |
| Blue (`#DCEEFB`) | New components that didn't exist before | `newc` |
| Gray (`#EAEAED`) | Context, data, existing code | `domain` |
| Pink (`#FFE8FB`) | Caution / being replaced | `caution` |
| Red tint (`#FFF0EB`) | Discard / danger | `discard` |
| Dark purple (`#4C0C3A`) | External / out of scope | `external` |

---

## Diagram 4: The Seams — Where We Cut

<!-- Diagram: Incision Points
     Reason: Reader needs to see exactly where in the existing code the abstraction boundary is drawn
     Aha: There are only 2 incision points in the entire codebase — rankContent() and modifyLiteStoreCollection() — and Phase 1 wraps everything behind each with a single RANK_ALL step -->

```mermaid
flowchart TB
    subgraph vert["  Vertical Ranking  "]
        direction TB
        V1("reOrderGlobalEntitiesV2()"):::domain
        V2("rankContent()"):::hero
        V3("getEntities → getScoreBundle<br>→ getBoostBundle<br>→ getRankableContent"):::domain
        V4("VerticalRankAllStep"):::newc

        V1 --> V2
        V2 -->|"old path"| V3
        V2 -.->|"UBP incision"| V4
        V4 -.->|"delegates to<br>same code"| V3
    end

    subgraph horiz["  Horizontal Ranking  "]
        direction TB
        H1("StoreRanker.rank()"):::domain
        H2("modifyLiteStoreCollection()"):::hero
        H3("scoredCollections()<br>→ when(rankingType)"):::domain
        H4("HorizontalRankAllStep"):::newc

        H1 --> H2
        H2 -->|"old path"| H3
        H2 -.->|"UBP incision"| H4
        H4 -.->|"delegates to<br>same code"| H3
    end

    classDef domain fill:#EAEAED,stroke:#B8B8C2,color:#505058,stroke-width:1px
    classDef hero fill:#FF3008,stroke:#D42807,color:#FFFFFF,stroke-width:1.5px
    classDef newc fill:#DCEEFB,stroke:#7BBCE0,color:#1A3A50,stroke-width:1px

    style vert fill:transparent,stroke:#C8C8D0,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
    style horiz fill:transparent,stroke:#C8C8D0,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
```

**Why each color:**
- **Red nodes** — `rankContent()` and `modifyLiteStoreCollection()` are the incision points. This is where we cut.
- **Blue nodes** — `VerticalRankAllStep` and `HorizontalRankAllStep` are the new components that wrap the legacy code.
- **Gray nodes** — existing call chain and legacy methods. Context — not the point.
- **Dashed edges** — the UBP path. Solid edges are the existing old path.

---

## Diagram 5: The Four Interfaces

<!-- Diagram: Interface Composition
     Reason: Reader needs to see how 4 interfaces compose into a complete ranking system
     Aha: Each interface has exactly one job — what gets ranked, how ranking works, infrastructure wrapping, orchestration — and they compose top-to-bottom without coupling -->

```mermaid
flowchart TB
    subgraph types["  9 Domain Types  "]
        direction LR
        T1("StoreCarousel"):::domain
        T2("ItemCarousel"):::domain
        T3("...7 more"):::domain
    end

    SC("Scorable<br>scorableId() · predictionScore<br>withPredictionScore()"):::hero

    RS("RankingStep&lt;S&gt;<br>execute(items, ctx)<br>→ List&lt;Scorable&gt;"):::newc

    RH("RankingHandler<br>BaseHandler · StepHandler<br>handle(items, ctx)"):::newc

    RK("Ranker&lt;S&gt;<br>rank(items, stepTypes, ctx)<br>buildChain → dispatch"):::newc

    T1 -->|"implements"| SC
    T2 -->|"implements"| SC
    T3 -->|"implements"| SC
    SC -->|"operated on by"| RS
    RS -->|"wrapped by"| RH
    RH -->|"chained by"| RK

    classDef domain fill:#EAEAED,stroke:#B8B8C2,color:#505058,stroke-width:1px
    classDef hero fill:#FF3008,stroke:#D42807,color:#FFFFFF,stroke-width:1.5px
    classDef newc fill:#DCEEFB,stroke:#7BBCE0,color:#1A3A50,stroke-width:1px

    style types fill:transparent,stroke:#C8C8D0,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
```

**Why each color:**
- **Red** — `Scorable` is the core abstraction this RFC proposes. Everything flows from it.
- **Blue** — `RankingStep`, `RankingHandler`, `Ranker` are new engine components. They didn't exist before.
- **Gray** — domain types are existing code. They add `override` and gain the interface — they're not the point.

---

## Diagram 6: Engine Dispatch — Handler Chain

<!-- Diagram: Engine Step Chain
     Reason: Reader needs to understand how Ranker.rank() dispatches work at runtime
     Aha: The engine doesn't change between Phase 1 and Phase 2 — only the step list passed in changes. One RANK_ALL step today, five granular steps tomorrow, same engine code. -->

```mermaid
flowchart TB
    RK("Ranker.rank()<br>buildChain(stepTypes)"):::hero

    subgraph p1["  Phase 1: stepTypes = [RANK_ALL] — shipped  "]
        H1("StepHandler<br>VerticalRankAllStep"):::newc
    end

    subgraph p2["  Phase 2: stepTypes = [SCORING, BOOST, DIVERSITY, POSITION, PINNING] — planned  "]
        direction LR
        S1("MODEL<br>SCORING"):::newc
        S2("MULTIPLIER<br>BOOST"):::newc
        S3("DIVERSITY<br>RERANK"):::newc
        S4("POSITION<br>BOOSTING"):::newc
        S5("FIXED<br>PINNING"):::newc
        S1 -->|".next"| S2 -->|".next"| S3 -->|".next"| S4 -->|".next"| S5
    end

    RK -->|"shipped"| H1
    RK -.->|"planned"| S1

    classDef domain fill:#EAEAED,stroke:#B8B8C2,color:#505058,stroke-width:1px
    classDef hero fill:#FF3008,stroke:#D42807,color:#FFFFFF,stroke-width:1.5px
    classDef newc fill:#DCEEFB,stroke:#7BBCE0,color:#1A3A50,stroke-width:1px

    style p1 fill:transparent,stroke:#A0CCE8,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
    style p2 fill:transparent,stroke:#A0CCE8,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
```

**Why each color:**
- **Red** — `Ranker.rank()` is the single entry point. The action. Same code in both phases.
- **Blue** — all step handlers are new. Phase 1 has one, Phase 2 has five. Engine unchanged.
- **Solid edge** — shipped path. **Dashed edge** — planned path.

---

## Diagram 7: Vertical + Horizontal Side by Side

<!-- Diagram: Mirror Architecture
     Reason: Reader needs to see that vertical and horizontal are literally the same architecture
     Aha: Same Scorable interface, same Ranker class, same RankingStep contract — only the step type enum and step implementation differ -->

```mermaid
flowchart LR
    subgraph vert["  Vertical Ranking  "]
        direction TB
        VI("PostProcessor<br>.rankContent()"):::domain
        VE("Ranker&lt;VerticalStepType&gt;"):::newc
        VS("VerticalRankAllStep"):::newc
        VSC("Scorable"):::hero
        VI --> VE
        VE --> VS
        VE --> VSC
    end

    subgraph horiz["  Horizontal Ranking  "]
        direction TB
        HI("StoreRanker<br>.modifyLiteStoreCollection()"):::domain
        HE("Ranker&lt;HorizontalStepType&gt;"):::newc
        HS("HorizontalRankAllStep"):::newc
        HSC("Scorable"):::hero
        HI --> HE
        HE --> HS
        HE --> HSC
    end

    classDef domain fill:#EAEAED,stroke:#B8B8C2,color:#505058,stroke-width:1px
    classDef hero fill:#FF3008,stroke:#D42807,color:#FFFFFF,stroke-width:1.5px
    classDef newc fill:#DCEEFB,stroke:#7BBCE0,color:#1A3A50,stroke-width:1px

    style vert fill:transparent,stroke:#A0CCE8,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
    style horiz fill:transparent,stroke:#A0CCE8,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
```

**Why each color:**
- **Red** — `Scorable` appears in both layers. Same interface. The core abstraction.
- **Blue** — `Ranker` and step implementations are new in both layers. Same class, different type parameter.
- **Gray** — incision points are existing code. They're where we wire in, not what we're building.

---

## Diagram 8: Design Patterns — What Replaces What

<!-- Diagram: Pattern Replacement
     Reason: Reader needs to see these aren't arbitrary interfaces — each maps to a well-known pattern
     Aha: Template Method (rigid inheritance skeleton) is replaced by four composable patterns — Interface Inheritance + Strategy + Chain of Responsibility + Facade -->

```mermaid
flowchart LR
    OLD("Template Method<br>BaseEntityRanker<br>Configuration.rank()<br>— rigid inheritance —"):::caution

    subgraph newp["  Config-Driven Composition  "]
        direction TB
        A("Interface Inheritance<br>→ Scorable"):::hero
        B("Strategy<br>→ RankingStep&lt;S&gt;"):::newc
        C("Chain of Responsibility<br>→ RankingHandler"):::newc
        D("Facade<br>→ Ranker.rank()"):::newc
        A --> B --> C --> D
    end

    OLD -->|"replaced by"| A

    classDef domain fill:#EAEAED,stroke:#B8B8C2,color:#505058,stroke-width:1px
    classDef hero fill:#FF3008,stroke:#D42807,color:#FFFFFF,stroke-width:1.5px
    classDef newc fill:#DCEEFB,stroke:#7BBCE0,color:#1A3A50,stroke-width:1px
    classDef caution fill:#FFE8FB,stroke:#E0A8DC,color:#4C0C3A,stroke-width:1px

    style newp fill:transparent,stroke:#A0CCE8,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
```

**Why each color:**
- **Pink** — Template Method is the old pattern being replaced. Caution — this is what's going away.
- **Red** — `Scorable` (Interface Inheritance) is the foundational action. Everything starts here.
- **Blue** — Strategy, Chain of Responsibility, Facade are the new engine patterns. They compose top-to-bottom.

---

## Diagram 9: Shadow Validation → Rollout

<!-- Diagram: Safe Delivery
     Reason: Reader needs confidence that this cannot break production
     Aha: The old path ALWAYS runs during shadow — users are never at risk. The shadow result is discarded. Rollback is instant — flip a DV, no deploy. -->

```mermaid
flowchart TB
    subgraph shadow["  Shadow Mode — DV: ubp_shadow_vertical_ranking  "]
        direction TB
        R1("Request"):::domain
        PP1("PostProcessor<br>.rankContent()"):::hero
        OLD1("Old Path<br>always runs"):::domain
        DV1{"shadow<br>enabled?"}:::domain
        NEW1("Ranker engine<br>parallel coroutine"):::newc
        RES1("Result → User"):::domain
        CMP("Compare sort orders<br>Log divergences<br>Discard shadow result"):::discard

        R1 --> PP1
        PP1 --> OLD1 --> RES1
        PP1 --> DV1
        DV1 -->|"Yes"| NEW1 --> CMP
        DV1 -->|"No"| RES1
    end

    subgraph rollout["  Rollout Mode — 1% → 5% → 25% → 50% → 100%  "]
        direction TB
        R2("Request"):::domain
        PP2("PostProcessor<br>.rankContent()"):::hero
        DV2{"rollout<br>enabled?"}:::domain
        NEW2("Ranker engine<br>primary"):::newc
        OLD2("Old Path<br>unchanged"):::domain
        RES2("Result → User"):::domain

        R2 --> PP2 --> DV2
        DV2 -->|"Yes"| NEW2 --> RES2
        DV2 -->|"No"| OLD2 --> RES2
    end

    shadow -.->|"divergence = 0<br>sustained"| rollout

    classDef domain fill:#EAEAED,stroke:#B8B8C2,color:#505058,stroke-width:1px
    classDef hero fill:#FF3008,stroke:#D42807,color:#FFFFFF,stroke-width:1.5px
    classDef newc fill:#DCEEFB,stroke:#7BBCE0,color:#1A3A50,stroke-width:1px
    classDef discard fill:#FFF0EB,stroke:#FF3008,color:#FF3008,stroke-width:1px

    style shadow fill:transparent,stroke:#E0A090,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
    style rollout fill:transparent,stroke:#A0CCE8,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
```

**Why each color:**
- **Red nodes** — `rankContent()` is the incision point. The action.
- **Blue nodes** — Ranker engine is the new component. In shadow mode it runs silently; in rollout it's primary.
- **Red-tinted node** — "Compare + Discard" is danger/discard. Shadow result is thrown away.
- **Gray** — old path, requests, results. Context that always exists.
- **Subgraph borders** — shadow has red-tinted border (risk-aware); rollout has blue-tinted (new path live).

---

## Diagram 10: The Full Pipeline — 4 Layers

<!-- Diagram: Pipeline Scope
     Reason: Reader needs to know what UBP changes vs what it doesn't touch
     Aha: UBP only governs Layers 3 and 4. Retrieval, grouping, and ICP ads blending are completely untouched. -->

```mermaid
flowchart TB
    REQ("Request"):::domain
    L1("Layer 1: Retrieval<br>Fetch candidates"):::domain
    L2("Layer 2: Grouping<br>Bucket into carousels"):::domain

    subgraph ubp3["  Layer 3: Horizontal — UBP scope  "]
        H("StoreRanker.rank()"):::hero
        HE("Ranker&lt;HorizontalStepType&gt;"):::newc
        H --> HE
    end

    subgraph ubp4["  Layer 4: Vertical — UBP scope  "]
        V("PostProcessor.rankContent()"):::hero
        VE("Ranker&lt;VerticalStepType&gt;"):::newc
        V --> VE
    end

    ADS("ICP Ads Blending<br>separate system"):::external
    SER("Serialize → Client"):::domain

    REQ --> L1 --> L2 --> H --> V --> SER
    H --> ADS --> SER

    classDef domain fill:#EAEAED,stroke:#B8B8C2,color:#505058,stroke-width:1px
    classDef hero fill:#FF3008,stroke:#D42807,color:#FFFFFF,stroke-width:1.5px
    classDef newc fill:#DCEEFB,stroke:#7BBCE0,color:#1A3A50,stroke-width:1px
    classDef external fill:#4C0C3A,stroke:#36082A,color:#FFFFFF,stroke-width:1px

    style ubp3 fill:transparent,stroke:#E0A090,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
    style ubp4 fill:transparent,stroke:#E0A090,stroke-width:1px,stroke-dasharray:6 4,color:#A0A0A8
```

**Why each color:**
- **Red nodes** — incision points where UBP wires in. The action.
- **Blue nodes** — new Ranker engines. The components this RFC proposes.
- **Gray nodes** — retrieval, grouping, serialize. Untouched. Context.
- **Dark purple node** — ICP Ads is external, out of scope, separate system.
- **Red-tinted subgraph borders** — Layers 3 and 4 are UBP scope. The reader's eye is drawn here.
