# POC: Article-Inspired Ranking & Scoring Patterns

Maps concepts from [Karishma Agrawal's feed ranking article](https://karishma-agr1996.medium.com/how-feed-ranking-algorithms-work-the-hidden-system-design-behind-personalized-feeds-4cd243fbd772) to GoF patterns. Complements `poc-generic-ranking.md` (pipeline engine) by focusing on **scoring models** and **re-ranking**.

## Patterns

| Pattern | Article Concept | Role |
|---|---|---|
| **Composite** | `Score = w₁P(like) + w₂P(comment) + w₃P(share) + w₄P(watch)` | Leaf scores aggregate into a composite tree |
| **Strategy** | Scoring models (LR, XGBoost, DNN) | Swap scoring algorithms without changing pipeline |
| **Chain of Responsibility** | Multi-pass ranking (Pass 0 → 1 → 2) | Each pass filters/re-scores, forwards survivors |
| **Decorator** | Re-ranking rules (diversity, freshness, integrity) | Stackable post-ranking constraints |

---

## Kotlin POC

```kotlin
// ── Data ────────────────────────────────────────────────────────

data class FeedItem(val id: String, val category: String, val ageMinutes: Int, val score: Double = 0.0)

// ── 1. Composite — Score Aggregation ────────────────────────────
// Leaf: single signal. Composite: weighted sum of children.
// Maps to: Score = w₁P(like) + w₂P(comment) + w₃P(share) + w₄P(watch)

sealed interface Score {
    fun value(item: FeedItem): Double
}

class LeafScore(private val name: String, private val fn: (FeedItem) -> Double) : Score {
    override fun value(item: FeedItem) = fn(item)
    override fun toString() = name
}

class CompositeScore(private val children: List<Pair<Double, Score>>) : Score {
    override fun value(item: FeedItem) =
        children.sumOf { (weight, score) -> weight * score.value(item) }
}

// ── 2. Strategy — Scoring Models ────────────────────────────────
// Each model is a strategy that assigns scores to items.
// Maps to: LR vs XGBoost vs DNN as interchangeable scoring backends.

fun interface ScoringModel {
    fun score(items: List<FeedItem>, scoreTree: Score): List<FeedItem>
}

// Logistic-regression-style: direct weighted sum
val linearModel = ScoringModel { items, tree ->
    items.map { it.copy(score = tree.value(it)) }
}

// DNN-style: same composite input, but applies non-linear transform
val dnnModel = ScoringModel { items, tree ->
    items.map {
        val raw = tree.value(it)
        it.copy(score = 1.0 / (1.0 + Math.exp(-raw)))  // sigmoid
    }
}

// ── 3. Chain of Responsibility — Multi-Pass Ranking ─────────────
// Pass 0: broad recall (thousands). Pass 1: ML scoring (hundreds).
// Pass 2: final top-K. Each pass filters/re-scores and forwards.

abstract class RankingPass(private val next: RankingPass? = null) {
    fun process(items: List<FeedItem>): List<FeedItem> {
        val result = apply(items)
        return next?.process(result) ?: result
    }
    protected abstract fun apply(items: List<FeedItem>): List<FeedItem>
}

// Pass 0: lightweight recall filter (simulate pre-filter by score threshold)
class RecallPass(private val minScore: Double, next: RankingPass? = null) : RankingPass(next) {
    override fun apply(items: List<FeedItem>) = items.filter { it.score >= minScore }
}

// Pass 1: ML scoring pass — apply model + score tree
class ScoringPass(
    private val model: ScoringModel,
    private val scoreTree: Score,
    next: RankingPass? = null,
) : RankingPass(next) {
    override fun apply(items: List<FeedItem>) = model.score(items, scoreTree)
}

// Pass 2: top-K selection
class TopKPass(private val k: Int, next: RankingPass? = null) : RankingPass(next) {
    override fun apply(items: List<FeedItem>) = items.sortedByDescending { it.score }.take(k)
}

// ── 4. Decorator — Re-Ranking Rules ─────────────────────────────
// Each decorator wraps the previous ranker and applies one constraint.
// Stack them: base → diversity → freshness → integrity.

fun interface ReRanker {
    fun rerank(items: List<FeedItem>): List<FeedItem>
}

// Base: identity (or final sort)
class BaseReRanker : ReRanker {
    override fun rerank(items: List<FeedItem>) = items.sortedByDescending { it.score }
}

// Decorator base
abstract class ReRankDecorator(protected val inner: ReRanker) : ReRanker

// Diversity: no more than N consecutive items from the same category
class DiversityDecorator(inner: ReRanker, private val maxConsecutive: Int = 2) : ReRankDecorator(inner) {
    override fun rerank(items: List<FeedItem>): List<FeedItem> {
        val base = inner.rerank(items)
        val result = mutableListOf<FeedItem>()
        val deferred = mutableListOf<FeedItem>()
        for (item in base) {
            val tail = result.takeLast(maxConsecutive)
            if (tail.size == maxConsecutive && tail.all { it.category == item.category }) {
                deferred += item
            } else {
                result += item
            }
        }
        return result + deferred
    }
}

// Freshness: boost items younger than threshold
class FreshnessDecorator(inner: ReRanker, private val maxAgeMinutes: Int = 60, private val boost: Double = 1.5) : ReRankDecorator(inner) {
    override fun rerank(items: List<FeedItem>): List<FeedItem> {
        val boosted = items.map {
            if (it.ageMinutes <= maxAgeMinutes) it.copy(score = it.score * boost) else it
        }
        return inner.rerank(boosted)
    }
}

// Integrity: remove items flagged by policy (simulated by ID prefix)
class IntegrityDecorator(inner: ReRanker) : ReRankDecorator(inner) {
    override fun rerank(items: List<FeedItem>) =
        inner.rerank(items.filter { !it.id.startsWith("flagged-") })
}

// ── main — Wire & Run ───────────────────────────────────────────

fun main() {
    // Composite score tree: 0.4·engagement + 0.3·relevance + 0.2·recency + 0.1·quality
    val engagement = LeafScore("engagement") { it.id.hashCode().and(0xFF) / 255.0 }
    val relevance  = LeafScore("relevance")  { if (it.category == "food") 0.9 else 0.4 }
    val recency    = LeafScore("recency")    { 1.0 - (it.ageMinutes / 1440.0).coerceIn(0.0, 1.0) }
    val quality    = LeafScore("quality")    { 0.7 }

    val scoreTree = CompositeScore(listOf(0.4 to engagement, 0.3 to relevance, 0.2 to recency, 0.1 to quality))

    // Sample items (pre-recall pool)
    val pool = listOf(
        FeedItem("post-1", "food", 10),
        FeedItem("post-2", "food", 30),
        FeedItem("post-3", "food", 120),
        FeedItem("post-4", "tech", 5),
        FeedItem("post-5", "tech", 200),
        FeedItem("post-6", "travel", 15),
        FeedItem("flagged-7", "food", 2),
        FeedItem("post-8", "travel", 500),
    )

    // Chain: scoring → recall filter → top-K
    val chain = ScoringPass(
        model = linearModel,
        scoreTree = scoreTree,
        next = RecallPass(
            minScore = 0.3,
            next = TopKPass(k = 5),
        ),
    )

    val ranked = chain.process(pool)

    // Decorator stack: integrity → freshness → diversity → base sort
    val reranker: ReRanker = DiversityDecorator(
        FreshnessDecorator(
            IntegrityDecorator(BaseReRanker())
        )
    )

    val final = reranker.rerank(ranked)

    println("=== Final Feed ===")
    final.forEachIndexed { i, item ->
        println("${i + 1}. [${item.category}] ${item.id} — score: ${"%.3f".format(item.score)}, age: ${item.ageMinutes}m")
    }
}
```

## How It Maps to the Article

```
Article Pipeline                     POC Pattern
─────────────────────────────────    ──────────────────────
Candidate Generation (Pass 0)   →   RecallPass (Chain)
ML Scoring (Pass 1)             →   ScoringPass (Chain) + ScoringModel (Strategy)
  └─ Score formula              →   CompositeScore (Composite)
Final Selection (Pass 2)        →   TopKPass (Chain)
Re-Ranking Rules                →   Decorator stack
  ├─ Diversity                  →   DiversityDecorator
  ├─ Freshness                  →   FreshnessDecorator
  └─ Integrity/Safety           →   IntegrityDecorator
```

## Relation to Existing POC

The generic ranking POC (`poc-generic-ranking.md`) defines the **pipeline engine** — `Scorable`, `RankingStep`, `RankingHandler`, `Ranker`. This POC zooms into the **scoring** and **re-ranking** layers that the engine delegates to. In production, `ScoringPass` would become a `RankingStep<VerticalStepType>` and the decorator stack would be a separate step or post-processing phase.
