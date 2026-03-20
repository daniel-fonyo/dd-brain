# Part 0A: Safe Refactoring Strategy

> How to ship FeedRow and RankingStep abstractions into feed-service without regressions.
> Based on Michael Feathers' *Working Effectively with Legacy Code* and the Strangler Fig pattern.

---

## Core Principle: Cover and Modify (Never Edit and Pray)

**Edit and Pray:** Understand the code, make changes, poke around to see if it broke. This is
how feed-service ranking changes work today.

**Cover and Modify:** Write tests that lock down current behavior. Make changes. If tests pass,
the refactoring preserved behavior. If they fail, you changed something — investigate or revert.

We use **Cover and Modify** for every change.

---

## What Is a Characterization Test?

From Feathers: *"A characterization test is a test that characterizes the actual behavior of a
piece of code."*

It does NOT test what the code **should** do. It tests what the code **actually does right now**.

| Dimension | Unit Test | Characterization Test |
|---|---|---|
| Purpose | Verify correctness against spec | Lock down existing behavior |
| When written | Before/alongside new code | Before modifying legacy code |
| Assertion source | Derived from requirements | Derived from running the code |
| Bugs | Test should catch them | Bugs are captured as-is — users depend on this behavior |
| Lifespan | Permanent | Temporary safety net — replaced after refactoring |
| Failure meaning | Code is wrong | Behavior changed (may or may not be intentional) |

### How to Write One

1. Pick the code you want to test
2. Write a test with a **deliberately wrong assertion** (e.g., `assertEquals(0, result)`)
3. Run the test — it fails and **shows the actual value**
4. Take that actual value and make it the expected value
5. Now you have a test documenting current behavior

This sounds backwards. That's the point. You don't know what the code does, so you ask it.

---

## The Order of Operations

```
Phase 0: SCRATCH REFACTORING (understand)
  │  Play with the code. Extract methods, rename variables, move things around.
  │  Goal: understand the code, not clean it.
  │  ⚠ REVERT ALL CHANGES WHEN DONE. Do not commit scratch refactoring.
  │
Phase 1: IDENTIFY SEAMS (find test entry points)
  │  A seam = a place where you can alter behavior without editing in that place.
  │  In Kotlin/Spring: constructor injection points, interfaces, Spring beans.
  │  Find where you can inject test doubles to isolate the code.
  │
Phase 2: BREAK DEPENDENCIES (make it testable)
  │  Minimal, careful changes to make the code testable:
  │  - Extract interface from a class you need to mock
  │  - Add constructor parameter for a dependency that's currently hardcoded
  │  - Use @TestConfiguration to swap Spring beans
  │  These are the ONLY code changes allowed before characterization tests exist.
  │
Phase 3: WRITE CHARACTERIZATION TESTS (lock down behavior)
  │  Two levels:
  │  a) Integration-level: full ranking pipeline input → output (golden master)
  │  b) Method-level: specific methods you're about to extract
  │  All tests must pass consistently before proceeding.
  │
Phase 4: EXTRACT ABSTRACTIONS (the actual work)
  │  Now safe to extract FeedRow, RankingStep, etc.
  │  Characterization tests must stay green after every extraction.
  │  If a test fails: stop, investigate, fix or revert.
  │
Phase 5: SHADOW VALIDATE (old path vs new path)
  │  Run both paths for the same request.
  │  Compare outputs. Log divergences.
  │  Users only see old path results.
  │  Target: divergence_count = 0.
  │
Phase 6: SWITCH + CLEAN UP
     Ramp traffic from old to new path.
     Replace characterization tests with proper unit tests.
     Delete old code paths.
```

---

## Seams in Feed-Service Ranking Code

A **seam** is where you can swap behavior without editing the code at that point. The
**enabling point** is where you decide which behavior to use.

### Object Seams (primary — via constructor injection)

Feed-service uses Spring `@Component` with constructor injection. Every constructor parameter
is a seam.

```kotlin
// DefaultHomePagePostProcessor has constructor params:
@Component
class DefaultHomePagePostProcessor(
    private val storeRanker: DefaultHomePageStoreRanker,  // ← seam
    private val entityScorer: EntityRankerConfiguration,  // ← seam
    private val blendingUtil: BlendingUtil,                // ← seam (if injected)
    ...
)
```

In tests, inject mocks or test doubles via these constructor params.

**Problem:** Some ranking logic uses Kotlin `object` singletons (e.g., `BlendingUtil`,
`PinnedCarouselUtil`). These are NOT seams — you can't swap them via injection.

**Fix:** For objects we need to characterize, either:
- Test at a higher level where the object is called (integration-level characterization)
- Use MockK's `mockkObject()` to mock the singleton in tests (last resort)

### Link Seams (via Spring configuration)

```kotlin
@TestConfiguration
class RankingTestConfig {
    @Bean
    fun entityScorer(): EntityRankerConfiguration = mockk<EntityRankerConfiguration>().also {
        every { it.score(any()) } returns knownScoreBundle()
    }
}
```

Swap entire Spring beans for test doubles using `@TestConfiguration` or `@Profile("test")`.

---

## Characterization Tests for Feed-Service Ranking

### Level 1: Integration — Full Pipeline Golden Master

Captures the entire vertical ranking output for a known input. This is the highest-value
safety net.

```kotlin
@SpringBootTest
class VerticalRankingCharacterizationTest {

    @Autowired lateinit var postProcessor: DefaultHomePagePostProcessor
    @MockkBean lateinit var sibylClient: SibylClient  // control external RPC

    @Test
    fun `vertical ranking produces stable sort order for fixture input`() {
        // Arrange: load a fixture request + mock Sibyl to return known scores
        val context = loadFixture("typical_homepage_request")
        every { sibylClient.predict(any()) } returns loadFixture("sibyl_scores")

        // Act: run the full ranking path
        val result = postProcessor.reOrderGlobalEntitiesV2(context)

        // Assert: compare sort order to golden master
        val sortOrder = result.elements.map { it.id to it.sortOrder }
        assertEquals(loadGoldenMaster("vertical_sort_order"), sortOrder)
    }
}
```

**Fixture files:**
```
src/test/resources/
  fixtures/
    typical_homepage_request.json     # ExploreContext with known experiment map
    sibyl_scores.json                 # Known Sibyl prediction response
  golden/
    vertical_sort_order.json          # Expected carousel sort order
```

### Level 2: Method — Specific Ranking Logic

Before extracting `getScoreBundle()` into `ModelScoringStep`, characterize `BlendingUtil.blendBundle()`:

```kotlin
class BlendingUtilCharacterizationTest {

    @Test
    fun `blendBundle with default config preserves score order`() {
        val input = listOf(
            scorableEntity(id = "c1", score = 0.9),
            scorableEntity(id = "c2", score = 0.3),
            scorableEntity(id = "c3", score = 0.7),
        )
        val config = loadFixture("default_blending_config")

        val result = BlendingUtil.blendBundle(input, config)

        // Deliberately wrong assertion first time → observe actual output → fix
        assertEquals(
            listOf("c1" to 0.9, "c3" to 0.7, "c2" to 0.3),
            result.map { it.id to it.score },
        )
    }

    @Test
    fun `blendBundle with NV boost config applies multiplier`() {
        val input = listOf(
            scorableEntity(id = "c1", score = 0.5, verticalId = 1),  // Rx
            scorableEntity(id = "c2", score = 0.4, verticalId = 10), // NV
        )
        val config = loadFixture("nv_boost_blending_config")

        val result = BlendingUtil.blendBundle(input, config)

        // NV carousel boosted above Rx despite lower base score
        assertEquals(listOf("c2", "c1"), result.map { it.id })
    }
}
```

### Handling Non-Determinism

| Source | Mitigation |
|---|---|
| Timestamps | Inject `Clock.fixed(...)` via Spring bean. Scrub timestamps in golden masters. |
| Request IDs | Scrub UUIDs with regex before comparison: `requestId.replace(uuidPattern, "<UUID>")` |
| Sibyl RPC | Mock at client level with `@MockkBean`. Return deterministic scores. |
| Experiment map | Fixture includes a fixed experiment map. No live DV resolution in tests. |
| Coroutine ordering | Pin coroutine dispatcher to single thread in tests. |
| Runtime JSON | Load from test resources, not from live runtime cache. |

### Fixture Factory Pattern

```kotlin
object RankingFixtures {
    fun exploreContext(
        experimentMap: Map<String, String> = mapOf("hp_vertical_blending" to "control"),
        requestId: String = "test-request-001",
        consumerId: Long = 12345L,
        pageType: String = "HOMEPAGE",
    ): ExploreContext = ExploreContext(
        experimentMap = experimentMap,
        requestId = requestId,
        // ... defaults for all other fields
    )

    fun scorableEntity(
        id: String = "carousel-1",
        score: Double = 0.5,
        verticalId: Int = 1,
        type: String = "STORE_CAROUSEL",
    ): ScorableEntityStoreCarousel = ScorableEntityStoreCarousel(
        id = id,
        predictionScore = score,
        // ... defaults
    )

    fun blendingConfig(
        boostMultipliers: List<BoostMultiplier> = emptyList(),
        defaultMultiplier: Double = 1.0,
    ): VerticalBlendingConfig = VerticalBlendingConfig(
        calibrationConfig = CalibrationConfig.DEFAULT,
        intentScoringConfig = IntentScoringConfig.DEFAULT,
        verticalBoostWeights = VerticalBoostWeights(boostMultipliers, defaultMultiplier),
        rerankingParams = RerankingParams.DISABLED,
    )
}
```

Override only what matters per test. Defaults match prod behavior.

---

## Applying This to Each Implementation Step

### Step 1: FeedRow Interface + Adapters

**Risk:** Zero — pure addition. No existing code modified.

**Characterization tests needed:** None for the adapters themselves (they're new code, write
normal unit tests). But write characterization tests for the **code that will eventually consume
FeedRow** — the ranking pipeline — so you have a safety net before Step 2.

**Order:**
1. Write integration-level characterization test for `reOrderGlobalEntitiesV2()` output
2. Write method-level characterization tests for `BlendingUtil.blendBundle()`,
   `BlendingUtil.rerankEntitiesWithDiversity()`, `PinnedCarouselUtil` methods
3. Ship FeedRow interface + adapters (pure addition, no old code touched)
4. Write unit tests for adapters (`toFeedRow()` + `applyBackTo()` round-trip)

### Step 2: RankingStep Extraction

**Risk:** Medium — extracting existing logic into step classes. Must preserve behavior exactly.

**Order:**
1. Characterization tests from Step 1 are already green
2. Extract `ModelScoringStep` — wraps `getScoreBundle()` (Sibyl gRPC + `BlendingUtil.blendBundle()`
   including diversity rerank — all one atomic call). Characterization tests must stay green.
3. Extract `BoostAndRankStep` — wraps `getBoostBundle()` + `getRankingBundle()` + `getRankableContent()`
   (score assignment to domain objects, position boosting, deal multiplier, pin vs flow sort order,
   reassembly — all one atomic flow). Characterization tests must stay green.
4. After each extraction: run characterization tests. Green → continue. Red → revert + investigate.

**Critical rule:** The step's `process()` method literally calls the same existing method. No
behavior change. The step is a wrapper, not a rewrite.

```kotlin
// BoostAndRankStep.process() wraps the EXISTING calls:
override suspend fun process(rows, context, params) {
    val typedParams = params as BoostAndRankParams
    // Same calls the old path makes — same methods, same logic
    val boostBundle = getBoostBundle(rows, typedParams)
    val rankingBundle = getRankingBundle(rows, typedParams)
    val rankableContent = getRankableContent(rows, boostBundle, rankingBundle)
    // Score assignment, position boosting, deal multiplier, pin vs flow sort, reassembly
    applyRankings(rows, rankableContent, typedParams)
}
```

### Step 3: Traffic Router

**Risk:** Low — new code. But must not affect non-UBP traffic.

**Tests:** Normal unit tests (not characterization). Test hash distribution, bucket assignment,
fallback to control.

### Step 4: Engine + Wiring

**Risk:** Medium — adds an `if/else` branch in PostProcessor.

**Order:**
1. Ship engine (pure addition, no existing code touched)
2. Add the `if/else` branch in PostProcessor — old path is the `else`, byte-for-byte unchanged
3. Run characterization tests with UBP flag OFF → must be green (old path unchanged)
4. Run shadow: both paths for same request, compare outputs, log divergences
5. `divergence_count = 0` → safe to ramp traffic

---

## The Strangler Fig Applied to UBP

```
                        BEFORE                              AFTER
                        ──────                              ─────

Request ──→ PostProcessor.reOrderGlobalEntitiesV2()
                │                                      Request ──→ PostProcessor
                ├── rankAndMergeContent()                            │
                │     ├── Sibyl scoring                              ├── if (ubpFlag):
                │     ├── BlendingUtil                               │     FeedRowRanker.rank()
                │     └── Boosting                                   │       ├── ModelScoringStep
                └── NonRankable fixups                               │       └── BoostAndRankStep
                                                                     │
                                                                     └── else:
                                                                           rankAndMergeContent()
                                                                           (old path, unchanged)
```

The old path is never deleted until the new path is proven at 100% traffic. Both coexist.

---

## Summary: The Complete Checklist

```
Before ANY code change in feed-service ranking:

□ Scratch refactoring done (and reverted) to understand the code
□ Integration-level characterization test for full pipeline output
□ Method-level characterization tests for each method being extracted
□ All characterization tests passing consistently (run 3x to check for flakiness)
□ Non-determinism handled (Clock, Sibyl mock, fixed experiment map)
□ Fixture factories created for complex domain objects

For each extraction:

□ Write/verify characterization test covers the code being changed
□ Extract into new interface/class
□ Run characterization tests → must be green
□ Write proper unit tests for the new abstraction
□ Ship behind flag (old path unchanged for non-flagged traffic)

For the switch:

□ Shadow validate: both paths, compare outputs, divergence_count = 0
□ Canary: 1% → 5% → 25% → 50% → 100%
□ Monitor: latency, error rates, ranking quality metrics
□ Replace characterization tests with proper unit/integration tests
□ Delete old code path
```
