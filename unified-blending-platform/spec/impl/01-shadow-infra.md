# Part 1: Shadow Infrastructure + Contract Assembly

## Goal

Run both the old ranking path and the new UBP engine on every shadow request. Always return
the old result to the user. **For each request, dynamically assemble the equivalent UBP
contract JSON from what the old path actually did** — log it so we can visually inspect what
the contract looks like for real traffic. Compare sort orders. Iterate on the contract shape
until divergence hits zero, then freeze into a static `control.json`.

## Why Contract Assembly (Not Static control.json First)

The previous approach tried to hand-author `control.json` from specs, then debug why it
didn't match prod. That's backwards — there are too many dynamic decisions in the old path
(experiment-dependent predictor selection, per-request config overrides, per-RankingType
when-chains) to get right from paper.

The new approach: **let prod behavior define the contract shape.**

```
OLD approach (too much ambiguity):
  design contract → implement → compare to prod → debug divergences

NEW approach (fast, visual):
  observe prod per-request → assemble equivalent UBP contract → log it → inspect → iterate
```

For every shadow request, the assembler constructs the UBP JSON that WOULD have been fed to
the engine. You can grep for any `request_id` and see exactly what the contract looked like.
Once the shape stabilizes across enough requests, freeze it into static `control.json`.

---

## Architecture

```
Shadow request
  │
  ├── 1. Run OLD path → result_old (returned to user)
  │       While running, old path records decisions (predictor chosen, config values used, etc.)
  │
  ├── 2. UbpContractAssembler.assemble(context, decisions)
  │       → assembledContract: UnifiedExperimentConfig JSON
  │       → LOGGED as structured JSON (queryable, visually inspectable)
  │
  ├── 3. Run UBP engine with assembledContract → result_new
  │
  └── 4. Compare result_old vs result_new → log divergences
```

The assembler reads the **same sources the old path reads** — same predictor selection logic,
same runtime JSON, same pinned carousel order. It just structures the output as UBP JSON.

---

## UbpContractAssembler

```kotlin
// libraries/common/.../ubp/shadow/UbpContractAssembler.kt
class UbpContractAssembler(
    private val p13nRuntimeUtil: P13nRuntimeUtil,
    private val pinnedCarouselUtil: PinnedCarouselUtil,
    private val collectionScorer: CollectionScorer,
) {
    /**
     * Observes what the old vertical path used for this request and constructs the
     * equivalent UBP JSON. The assembled contract is what control.json SHOULD contain
     * to replicate this request's ranking.
     */
    fun assembleVertical(context: RankingContext): AssembledContract {
        // 1. What predictor did the old path use for this request?
        val predictorName = collectionScorer.getPredictorName(context)
        val modelName = collectionScorer.getModelName(context)

        // 2. What blending config was applied?
        val blendingConfig = p13nRuntimeUtil.getVerticalBlendingConfig(context)

        // 3. What pinned carousel order was used?
        val pinnedOrder = pinnedCarouselUtil.getPinnedCarouselOrder(context)

        // 4. Assemble the equivalent UBP JSON
        val contract = UnifiedExperimentConfig(
            treatments = mapOf("control" to TreatmentConfig(
                predictors = mapOf("p_act" to PredictorConfig(
                    predictorName = predictorName.label,
                    modelName = modelName,
                    useCase = "USE_CASE_HOME_PAGE_VERTICAL_RANKING",
                )),
                verticalPipeline = PipelineConfig(steps = listOf(
                    StepConfig("score", "MODEL_SCORING",
                        paramNode("predictor_ref", "p_act")),
                    StepConfig("blend", "MULTIPLIER_BOOST",
                        OBJECT_MAPPER_RELAX.valueToTree(MultiplierBoostParams(
                            calibrationConfig = blendingConfig.calibrationConfig,
                            intentScoringConfig = blendingConfig.intentScoringConfig,
                            verticalBoostWeights = blendingConfig.verticalBoostWeights,
                        ))),
                    StepConfig("diversity", "DIVERSITY_RERANK",
                        OBJECT_MAPPER_RELAX.valueToTree(DiversityRerankParams(
                            enabled = blendingConfig.rerankingParams.enabled,
                            diversityScoringParams = blendingConfig.rerankingParams.diversityScoringParams,
                        ))),
                    StepConfig("pin", "FIXED_PINNING",
                        OBJECT_MAPPER_RELAX.valueToTree(FixedPinningParams(
                            rules = pinnedOrder.map { PinRule(rowId = it.id, position = it.position) },
                        ))),
                )),
                outputConfig = OutputConfig(emitTrace = true),
            ))
        )
        return AssembledContract(layer = "vertical", contract = contract)
    }

    /**
     * Observes what the old horizontal path used for a specific carousel and constructs
     * the equivalent UBP JSON. Called once per carousel per request.
     */
    fun assembleHorizontal(
        context: RankingContext,
        collection: LiteStoreCollection,
    ): AssembledContract {
        val rankingType = collection.rankingType

        // Map the modifyLiteStoreCollection() when-chain to UBP steps
        val steps = when (rankingType) {
            RankingType.CAROUSEL,
            RankingType.NEW_VERTICAL_CAROUSEL -> listOf(
                StepConfig("score", "MODEL_SCORING", assembleHorizontalScoringParams(context, collection)),
                StepConfig("boost", "MULTIPLIER_BOOST", assembleHorizontalBoostParams(context, collection)),
                StepConfig("rules", "BUSINESS_RULES_SORT", paramNode("apply_open_closed", true)),
            )
            RankingType.PICKUP_STORE_FEED,
            RankingType.MAP_CAROUSEL -> listOf(
                StepConfig("rules", "BUSINESS_RULES_SORT", paramNode("sort_by", "DISTANCE")),
            )
            RankingType.BOOKMARKS -> listOf(
                StepConfig("rules", "BUSINESS_RULES_SORT", paramNode("sort_by", "SAVELIST_ORDER")),
            )
            RankingType.WHOLE_ORDER_REORDER -> listOf(
                StepConfig("score", "MODEL_SCORING", assembleHorizontalScoringParams(context, collection)),
                StepConfig("order_rerank", "ORDER_HISTORY_RERANK", paramNode("enabled", true)),
            )
            RankingType.NOOP -> emptyList()
            else -> listOf(
                StepConfig("score", "MODEL_SCORING", assembleHorizontalScoringParams(context, collection)),
                StepConfig("rules", "BUSINESS_RULES_SORT", paramNode("apply_open_closed", true)),
            )
        }

        val contract = UnifiedExperimentConfig(
            treatments = mapOf("control" to TreatmentConfig(
                horizontalPipeline = PipelineConfig(steps = steps),
                outputConfig = OutputConfig(emitTrace = true),
            ))
        )
        return AssembledContract(
            layer = "horizontal",
            contract = contract,
            carouselId = collection.id,
            rankingType = rankingType.name,
        )
    }
}

data class AssembledContract(
    val layer: String,                       // "vertical" or "horizontal"
    val contract: UnifiedExperimentConfig,
    val carouselId: String? = null,          // horizontal only
    val rankingType: String? = null,         // horizontal only
)
```

---

## What Gets Logged

Every shadow request emits the assembled contract as a structured log line. This is the
"aha moment" — you can grep for any request and see the exact contract.

### Vertical log example

```
ubp_assembled_contract
  layer=vertical
  request_id=req-abc123
  predictor_name=feed_ranking_fw
  model_name=store_ranker_fw_v3
  contract={
    "control": {
      "predictors": {
        "p_act": {
          "predictor_name": "feed_ranking_fw",
          "model_name": "store_ranker_fw_v3",
          "use_case": "USE_CASE_HOME_PAGE_VERTICAL_RANKING"
        }
      },
      "vertical_pipeline": {
        "steps": [
          { "id": "score", "type": "MODEL_SCORING", "params": { "predictor_ref": "p_act" } },
          {
            "id": "blend", "type": "MULTIPLIER_BOOST",
            "params": {
              "calibration_config": { "calibration_entries": [...], "default_multiplier": 1.0 },
              "intent_scoring_config": { "feature_configs": [...], "score_lookup_entries": [...], "default_score": 1.0 },
              "vertical_boost_weights": {
                "boosting_multipliers": [{ "vertical_ids": [2], "multiplier": 1.2 }],
                "default_multiplier": 1.0,
                "item_carousel_boosting_multipliers": [{ "vertical_ids": [10], "multiplier": 0.9 }],
                "item_carousel_default_multiplier": 1.0
              }
            }
          },
          { "id": "diversity", "type": "DIVERSITY_RERANK", "params": { "enabled": false } },
          { "id": "pin", "type": "FIXED_PINNING", "params": { "rules": [
            { "row_id": "taste_of_dashpass", "position": 3 },
            { "row_id": "member_pricing", "position": 0 }
          ] } }
        ]
      }
    }
  }
```

### Horizontal log example (per carousel)

```
ubp_assembled_contract
  layer=horizontal
  request_id=req-abc123
  carousel_id=favorites
  ranking_type=CAROUSEL
  contract={
    "control": {
      "horizontal_pipeline": {
        "steps": [
          { "id": "score", "type": "MODEL_SCORING", "params": { "predictor_name": "feed_ranking", "model_name": "store_ranker_v1" } },
          { "id": "boost", "type": "MULTIPLIER_BOOST", "params": { ... } },
          { "id": "rules", "type": "BUSINESS_RULES_SORT", "params": { "apply_open_closed": true } }
        ]
      }
    }
  }

ubp_assembled_contract
  layer=horizontal
  request_id=req-abc123
  carousel_id=pickup_nearby
  ranking_type=PICKUP_STORE_FEED
  contract={
    "control": {
      "horizontal_pipeline": {
        "steps": [
          { "id": "rules", "type": "BUSINESS_RULES_SORT", "params": { "sort_by": "DISTANCE" } }
        ]
      }
    }
  }
```

**What this reveals visually:**
- Different carousels get different step sequences — the when-chain is now visible as config
- The predictor can vary per request based on active experiments
- Boost weights and calibration values are the actual runtime values, not guesses
- Pin rules show exactly which carousels are pinned where

---

## Vertical Shadow (Updated)

```kotlin
// In DefaultHomePagePostProcessor.reOrderGlobalEntitiesV2()
val shadowMode = discoveryExperimentManager.getUbpVerticalShadowMode(context.experimentMap)

val (content, placementEventData) = when (shadowMode) {
    "shadow" -> {
        val oldResult = rankAndDedupeContent(context, ...)

        runCatching {
            // Assemble the contract from what the old path actually used
            val assembled = contractAssembler.assembleVertical(context)

            // Log it — this is the visual inspection point
            logger.info(
                "ubp_assembled_contract layer=vertical " +
                "request_id=${context.requestId} " +
                "predictor_name=${assembled.contract.predictorSummary()} " +
                "contract=${OBJECT_MAPPER_RELAX.writeValueAsString(assembled.contract)}"
            )

            // Run UBP engine with the assembled contract
            val rows = toFeedRows(layout)
            val resolved = runtimeUtil.resolveFromConfig(assembled.contract, "control")
            val newResult = feedRowRanker.rank(rows, resolved, context)

            // Compare
            val oldOrder = oldResult.first.toSortOrderMap()
            val newOrder = newResult.toSortOrderMap()
            val divergences = oldOrder.filter { (id, pos) -> newOrder[id] != pos }

            if (divergences.isNotEmpty()) {
                logger.warn(
                    "ubp_shadow_divergence layer=vertical " +
                    "request_id=${context.requestId} " +
                    "divergence_count=${divergences.size} " +
                    "diffs=${divergences.map { (id, old) -> "$id:$old->${newOrder[id]}" }.joinToString(",")}"
                )
            } else {
                logger.info("ubp_shadow_match layer=vertical request_id=${context.requestId}")
            }
        }.onFailure {
            logger.warn("ubp_shadow_error layer=vertical request_id=${context.requestId}", it)
        }

        oldResult
    }
    else -> rankAndDedupeContent(context, ...)
}
```

---

## Horizontal Shadow (Updated)

```kotlin
// In DefaultHomePageStoreRanker.rank()
val shadowMode = discoveryExptManager.getUbpHorizontalShadowMode(context.experimentMap)

return scoredCollections.map { collection ->
    CoroutineScope(...).async {
        val oldResult = modifyLiteStoreCollection(context, collection, storesById, metadata, ...)

        if (shadowMode == "shadow") {
            runCatching {
                // Assemble per-carousel contract
                val assembled = contractAssembler.assembleHorizontal(context, collection)

                logger.info(
                    "ubp_assembled_contract layer=horizontal " +
                    "request_id=${context.requestId} " +
                    "carousel_id=${collection.id} " +
                    "ranking_type=${assembled.rankingType} " +
                    "contract=${OBJECT_MAPPER_RELAX.writeValueAsString(assembled.contract)}"
                )

                val resolved = runtimeUtil.resolveFromConfig(assembled.contract, "control")
                val newResult = rowItemRanker.rank(collection, resolved, context)

                val divergences = oldResult.storeIds.zip(newResult.storeIds)
                    .filter { (old, new) -> old != new }

                if (divergences.isNotEmpty()) {
                    logger.warn(
                        "ubp_shadow_divergence layer=horizontal " +
                        "request_id=${context.requestId} " +
                        "carousel_id=${collection.id} " +
                        "ranking_type=${assembled.rankingType} " +
                        "divergence_count=${divergences.size}"
                    )
                }
            }.onFailure {
                logger.warn("ubp_shadow_error layer=horizontal request_id=${context.requestId}", it)
            }
        }

        oldResult
    }
}.awaitAll()
```

---

## The Convergence Path

```
Phase A: Assembly + Visual Inspection
  Shadow assembles contracts per request, logs them.
  You grep/query the logs, inspect contract JSONs visually.
  Identify: "the shape is stable — vertical always has 4 steps,
  horizontal has 3 default + per-carousel overrides."

Phase B: Freeze into Static control.json
  Once the shape stabilizes, take a representative assembled contract
  and write it as the static control.json.
  Switch shadow to use static control.json instead of assembler.

Phase C: Validate Static control.json
  Shadow with static control.json. Target: divergence_count = 0.
  Any divergence = the static contract drifted from the dynamic one.

Phase D: Promote to Real Traffic
  confidence = divergence_count == 0 for N hours on X% of traffic.
```

Phase A is where you spend time. Phases B-D are mechanical.

---

## New DV Manifest Entries

```kotlin
UBP_HP_VERTICAL_SHADOW("ubp_hp_vertical_shadow"),
UBP_HP_HORIZONTAL_SHADOW("ubp_hp_horizontal_shadow"),
```

Both default to `"off"`. Enable `"shadow"` for a canary % of traffic.

---

## Critical Rules

1. **New path errors must never affect users.** Wrapped in `runCatching`. On failure: log `ubp_shadow_error`, return old result.
2. **New path latency must not add to p99.** Log duration; alert if exceeds 5ms p99 additional.
3. **Shadow + assembler are temporary.** Delete once static control.json is validated. Don't let them become permanent scaffolding.
4. **Assembled contracts must be valid UBP JSON.** If the assembler produces JSON that the engine can't parse, that's a contract spec bug — fix the spec, not the assembler.

---

## Files to Create / Modify

| Action | File |
|---|---|
| New | `libraries/common/.../ubp/shadow/UbpContractAssembler.kt` |
| Modify | `DefaultHomePagePostProcessor.kt` — add shadow branch with assembler |
| Modify | `DefaultHomePageStoreRanker.kt` — add shadow branch with assembler |
| Modify | `DiscoveryExperimentManager.kt` — add 2 DV entries |

---

## Prerequisites

None. This is the first thing built. The assembler is standalone — it reads existing runtime
utils that already exist. The `FeedRowRanker` reference will be a stub until Parts 2-5 are
complete, but the **assembler can run immediately** and start logging contract JSONs even
before the engine exists. This is the fast path to visual feedback.

---

## Done When

- Assembler runs on canary traffic and emits `ubp_assembled_contract` logs for both layers
- You can grep a `request_id` and see the full vertical + horizontal contract JSONs
- Vertical contracts show: predictor chosen, blending params, diversity state, pin rules
- Horizontal contracts show: per-carousel step sequences, ranking type mappings
- Contract shape has stabilized across 1000+ requests
- Static `control.json` written from representative assembled contract
- Shadow with static `control.json` shows `divergence_count = 0`
