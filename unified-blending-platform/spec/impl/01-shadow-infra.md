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

## Ranking Input Inventory

A thorough audit of the vertical and horizontal ranking code paths identified every external
input that changes ranking behavior. The assembler must capture all of these.

### Vertical Ranking Inputs

| # | Input | Source | File | Assembler must capture |
|---|---|---|---|---|
| V1 | **Sibyl predictor + model name** | DV chain → `dv_treatment_sibyl_model_mapping.json` → `sibyl_model_config.json` | `StoreCollectionScorer.kt` | Final predictor/model name |
| V2 | **Blending config ID** | DV `hp_vertical_blending_config` | `P13nExperimentManager.kt` | Which config was selected |
| V3 | **Blending config values** | `hp_vertical_blending_config.json` (+ override file) | `P13nRuntimeUtil.kt` | calibration, intent, boost weights |
| V4 | **Diversity/reranking params** | Same blending config | `VerticalBlending.kt` | enabled, weight, coefficients |
| V5 | **Pinned carousel order** | `pinned_carousel_ranking_order.json` | `PinnedCarouselUtil.kt` | Pin rules |
| V6 | **Deal carousel score multiplier** | DV `deal_for_you_carousel_score_multiplier` | `Boosting.kt:164` | Multiplier value |
| V7 | **Boost-by-position allowlist** | `boost_by_position_carousel_id_allow_list.json` + Tiger Team `CONTENT_SYSTEM_BOOSTING` | `Boosting.kt:86-98`, `PersonalizationRuntimeUtil.kt` | Allowlist + enforcement logic |
| V8 | **NV unpinning** | DVs `enable_homepage_vertical_blender` + `should_unpin_nv_carousels` | `Boosting.kt:132` | Unpin state |
| V9 | **Reels UR integration** | DV `enableReelsUniversalRankerIntegration` | `DefaultHomePagePostProcessor.kt:192` | Whether reels scored through UR |
| V10 | **Full experimentMap** | `context.experimentMap` | All files | All active DVs for this request |

### Horizontal Ranking Inputs

| # | Input | Source | File | Assembler must capture |
|---|---|---|---|---|
| H1 | **Sibyl model per RankingType** | `dv_treatment_sibyl_model_mapping.json` per ranking type label | `StoreCollectionScorer.kt:99-130` | Model name per carousel |
| H2 | **Score modifier maps** | `store_ranker_boost_multiplier_by_business.json` + `by_store_id.json` | `PersonalizationRuntimeUtil.kt`, `StoreCollectionScorer.kt:339` | Per-business/store multipliers |
| H3 | **Boosting gates** | DVs `ENABLE_BOOSTING`, `SHOULD_DISABLE_BOOSTING`, `shouldBoostBusinessForStoreRanker` | `StoreCollectionScorer.kt:338` | Whether boost multipliers applied |
| H4 | **Business sort order overrides** | `business_sort_order_override_by_carousel_id.json` | `DiscoveryUtilsRuntimeUtil.kt`, `DefaultHomePageStoreRanker.kt:88` | Priority businesses per carousel |
| H5 | **Campaign sort orders** | `CampaignFeedResponse.placements` | `StoreCarouselService.kt:489-550` | Campaign-positioned stores |
| H6 | **Campaign eligibility gate** | DV `ENABLE_HOMEPAGE_CAMPAIGN_STORE_CAROUSEL_ELIGIBILITY` | `StoreCarouselService.kt:820` | Carousel store filtering |
| H7 | **AlwaysOpen ranking type** | `context.exploreLayoutContext.alwaysOpenType` | `DefaultHomePageStoreRanker.kt:180-221` | CONTROL vs SCHEDULE_AHEAD_AFTER_ASAP |
| H8 | **MX logo priorities** | `mx_logo_carousel_priority_biz_ids.json` + `priority_verticals.json` | `StoreCarouselService.kt:560-593` | Priority business/vertical reordering |
| H9 | **Comparator chain** | `StoreStatusComparator` → `ShipAnywhereComparator` → `StoreListComparator` → `StoreSortOrderMapComparator` | `StoreCarouselService.kt:730-744` | Full comparator order |
| H10 | **FW predictor selection** | DVs `shouldUseSRFWPredictorForStoreRanker`, `shouldUseNVCarouselFWPredictor` | `StoreCollectionScorer.kt:519-528` | Which model variant was used |
| H11 | **Feature set selection** | DVs `shouldUseSurfaceAwareFeatures*`, `shouldUseHorizontalPositionAsFeature` | `StoreCollectionScorer.kt:525-859` | Which features sent to Sibyl |
| H12 | **Whole order reorder thresholds** | `WholeOrderReorderRuntimeUtil` configs | `WholeOrderReorderFilterUtil.kt` | Min frequency, partial match, recency |
| H13 | **Taste carousel context** | `getTaste(experimentMap, filter)` | `ContextualStoreScorer.kt` | Which taste query passed to scorer |
| H14 | **Flagship store gate** | DV `ENABLE_FLAGSHIP_VLP` | `DefaultHomePageStoreRanker.kt:187` | Flagship prioritization |
| H15 | **Full experimentMap** | `context.experimentMap` | All files | All active DVs for this request |

### Runtime JSON Files (All Read via DeserializedRuntimeCache)

| File path | What it contains | Used by |
|---|---|---|
| `ranking/hp_vertical_blending_config.json` | Calibration, intent scoring, boost weights, diversity params per config ID | Vertical blending |
| `ranking/override_hp_vertical_blending_config.json` | Override configs (takes precedence if configId matches) | Vertical blending |
| `ranking/dv_treatment_sibyl_model_mapping.json` | DV→model name mapping per ranking type | Both layers |
| `sibyl_model_config.json` | Sibyl predictor registry (predictor name, shard, use case) | Both layers |
| `carousels/pinned_carousel_ranking_order.json` | Pinned carousel positions | Vertical pinning |
| `boost_by_position_carousel_id_allow_list.json` | Carousel IDs eligible for boost-by-position | Vertical boosting |
| `retail/store_ranker_boost_multiplier_by_business.json` | Score multiplier per business ID | Horizontal boosting |
| `retail/store_ranker_boost_multiplier_by_store_id.json` | Score multiplier per store ID (takes precedence) | Horizontal boosting |
| `carousels/business_sort_order_override_by_carousel_id.json` | Priority business IDs per carousel | Horizontal sort |
| `carousels/mx_logo_carousel_priority_verticals.json` | Vertical priority for MX logo carousels | Horizontal MX logo |
| `carousels/mx_logo_carousel_priority_biz_ids.json` | Business priority for MX logo carousels | Horizontal MX logo |
| `ranking/hp_nv_carousel_separation_shard.json` | NV/HR separation sharding | Vertical path routing |
| `ranking/hp_nv_carousel_override_models.json` | Model ID overrides for NV carousels | Vertical scoring |

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
  │       → experimentMap snapshot
  │       → LOGGED as structured JSON (queryable, visually inspectable)
  │
  ├── 3. Run UBP engine with assembledContract → result_new
  │
  └── 4. Compare result_old vs result_new → log divergences
```

The assembler reads the **same sources the old path reads** — same predictor selection logic,
same runtime JSON, same pinned carousel order, same boosting logic. It structures the output
as UBP JSON.

---

## UbpContractAssembler

```kotlin
// libraries/common/.../ubp/shadow/UbpContractAssembler.kt
class UbpContractAssembler(
    private val p13nRuntimeUtil: P13nRuntimeUtil,
    private val p13nExperimentManager: P13nExperimentManager,
    private val pinnedCarouselUtil: PinnedCarouselUtil,
    private val collectionScorer: CollectionScorer,
    private val personalizationRuntimeUtil: PersonalizationRuntimeUtil,
    private val discoveryExperimentUtil: DiscoveryExperimentUtil,
    private val discoveryUtilsRuntimeUtil: DiscoveryUtilsRuntimeUtil,
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
        val configId = p13nExperimentManager.getHpVerticalBlendingConfigID(context.experimentMap)
        val blendingConfig = p13nRuntimeUtil.getVerticalBlendingConfig(context)

        // 3. What pinned carousel order was used?
        val pinnedOrder = pinnedCarouselUtil.getPinnedCarouselOrder(context)

        // 4. What boosting state was active?
        val dealMultiplier = discoveryExperimentUtil.getDealCarouselScoreMultiplier(context.experimentMap)
        val boostAllowList = personalizationRuntimeUtil.boostByPositionCarouselIdAllowList()
        val nvUnpinEnabled = p13nExperimentManager.enableHomepageVerticalBlending(context.experimentMap)
            && p13nExperimentManager.shouldUnpinNvCarousels(context.experimentMap)

        // 5. Assemble the equivalent UBP JSON
        val steps = mutableListOf(
            StepConfig("score", "MODEL_SCORING",
                OBJECT_MAPPER_RELAX.valueToTree(ModelScoringParams(
                    predictorRef = "p_act",
                    predictorName = predictorName.label,
                    modelName = modelName,
                    calibrationConfig = blendingConfig.calibrationConfig,
                    intentScoringConfig = blendingConfig.intentScoringConfig,
                    verticalBoostWeights = blendingConfig.verticalBoostWeights,
                    diversityEnabled = blendingConfig.rerankingParams.enabled,
                    diversityScoringParams = blendingConfig.rerankingParams.diversityScoringParams,
                ))),
            StepConfig("boost_and_rank", "BOOST_AND_RANK",
                OBJECT_MAPPER_RELAX.valueToTree(BoostAndRankParams(
                    boostByPositionEnabled = boostAllowList.isNotEmpty(),
                    dealCarouselScoreMultiplier = dealMultiplier,
                    boostByPositionAllowList = boostAllowList.toList(),
                ))),
        )

        val contract = UnifiedExperimentConfig(
            treatments = mapOf("control" to TreatmentConfig(
                predictors = mapOf("p_act" to PredictorConfig(
                    predictorName = predictorName.label,
                    modelName = modelName,
                    useCase = "USE_CASE_HOME_PAGE_VERTICAL_RANKING",
                )),
                verticalPipeline = PipelineConfig(steps = steps),
                outputConfig = OutputConfig(emitTrace = true),
            ))
        )
        return AssembledContract(
            layer = "vertical",
            contract = contract,
            metadata = mapOf(
                "blending_config_id" to configId,
                "deal_multiplier" to dealMultiplier.toString(),
                "boost_allowlist_size" to boostAllowList.size.toString(),
                "nv_unpin_enabled" to nvUnpinEnabled.toString(),
            ),
        )
    }

    /**
     * Observes what the old horizontal path used for a specific carousel and constructs
     * the equivalent UBP JSON. Called once per carousel per request.
     */
    fun assembleHorizontal(
        context: RankingContext,
        collection: LiteStoreCollection,
        metadata: StoreRankerMetadata,
    ): AssembledContract {
        val rankingType = collection.rankingType

        // Capture scoring model for this carousel
        val scoringParams = assembleScoringParams(context, collection)

        // Capture score modifier maps (boost by business/store ID)
        val boostEnabled = p13nExperimentManager.shouldBoostBusinessForStoreRanker(context.experimentMap)
        val businessBoosts = if (boostEnabled) personalizationRuntimeUtil.getStoreRankerBoostMultipliersByBusiness() else emptyMap()
        val storeBoosts = if (boostEnabled) personalizationRuntimeUtil.getStoreRankerBoostMultipliersByStoreId() else emptyMap()

        // Capture business sort order overrides for this carousel
        val businessSortOverrides = discoveryUtilsRuntimeUtil
            .getBusinessSortOrderOverrideByCarouselId()[collection.id] ?: emptyList()

        // Capture campaign sort orders
        val campaignSortOrders = metadata.campaigns
            ?.placements?.get(collection.id)?.stores ?: emptyList()

        // Capture AlwaysOpen state
        val alwaysOpenType = context.exploreLayoutContext.alwaysOpenType

        // Map the modifyLiteStoreCollection() when-chain to UBP steps
        // Two coarse steps: MODEL_SCORING wraps scoredCollections() (Sibyl + score modifiers, atomic)
        //                    RANKING_SORT wraps modifyLiteStoreCollection() when(rankingType) dispatch
        val steps = when (rankingType) {
            RankingType.CAROUSEL,
            RankingType.NEW_VERTICAL_CAROUSEL,
            RankingType.CONTEXTUAL_TASTE_CAROUSEL,
            RankingType.INFLATION_MULTIPLIER_CAROUSEL,
            RankingType.RETENTION_RECOMMENDED_CAROUSEL -> listOf(
                StepConfig("score", "MODEL_SCORING", scoringParams),
                StepConfig("rank", "RANKING_SORT",
                    OBJECT_MAPPER_RELAX.valueToTree(RankingSortParams(
                        rankingType = rankingType.name,
                        boostEnabled = boostEnabled,
                        businessMultipliers = businessBoosts,
                        storeMultipliers = storeBoosts,
                        campaignStoreOrder = campaignSortOrders.map { it.storeId },
                        businessSortOverrides = businessSortOverrides,
                        applyOpenClosed = true,
                        alwaysOpenType = alwaysOpenType,
                    ))),
            )
            RankingType.MX_LOGO_CAROUSEL -> {
                val priorityBizIds = personalizationRuntimeUtil
                    .getMxLogoPriorityBizIds()[collection.id] ?: emptyList()
                val priorityVerticalIds = personalizationRuntimeUtil
                    .getMxLogoPriorityVerticalIds()[collection.id] ?: emptyList()
                listOf(
                    StepConfig("score", "MODEL_SCORING", scoringParams),
                    StepConfig("rank", "RANKING_SORT",
                        OBJECT_MAPPER_RELAX.valueToTree(RankingSortParams(
                            rankingType = rankingType.name,
                            applyOpenClosed = true,
                            alwaysOpenType = alwaysOpenType,
                            priorityBusinessIds = priorityBizIds,
                            priorityVerticalIds = priorityVerticalIds,
                        ))),
                )
            }
            RankingType.PICKUP_STORE_FEED,
            RankingType.MAP_CAROUSEL -> listOf(
                StepConfig("rank", "RANKING_SORT",
                    OBJECT_MAPPER_RELAX.valueToTree(RankingSortParams(
                        rankingType = rankingType.name,
                        sortBy = "DISTANCE",
                    ))),
            )
            RankingType.BOOKMARKS -> listOf(
                StepConfig("rank", "RANKING_SORT",
                    OBJECT_MAPPER_RELAX.valueToTree(RankingSortParams(
                        rankingType = rankingType.name,
                        sortBy = "SAVELIST_ORDER",
                    ))),
            )
            RankingType.DISTANCE_DECAYED_POPULARITY,
            RankingType.CATEGORY_AND_DISTANCE_DECAYED_POPULARITY -> listOf(
                StepConfig("rank", "RANKING_SORT",
                    OBJECT_MAPPER_RELAX.valueToTree(RankingSortParams(
                        rankingType = rankingType.name,
                        sortBy = "DISTANCE_DECAYED_POPULARITY",
                    ))),
            )
            RankingType.WHOLE_ORDER_REORDER -> listOf(
                StepConfig("score", "MODEL_SCORING", scoringParams),
                StepConfig("rank", "RANKING_SORT",
                    OBJECT_MAPPER_RELAX.valueToTree(RankingSortParams(
                        rankingType = rankingType.name,
                        orderHistoryRerankEnabled = true,
                    ))),
            )
            RankingType.NOOP -> emptyList()
            else -> listOf(
                StepConfig("score", "MODEL_SCORING", scoringParams),
                StepConfig("rank", "RANKING_SORT",
                    OBJECT_MAPPER_RELAX.valueToTree(RankingSortParams(
                        rankingType = rankingType.name,
                        applyOpenClosed = true,
                        alwaysOpenType = alwaysOpenType,
                    ))),
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
            metadata = mapOf(
                "boost_enabled" to boostEnabled.toString(),
                "business_overrides_count" to businessSortOverrides.size.toString(),
                "campaign_stores_count" to campaignSortOrders.size.toString(),
                "always_open_type" to alwaysOpenType,
            ),
        )
    }

    /** Assembles the Sibyl scoring params for this carousel based on active DVs + model mapping. */
    private fun assembleScoringParams(
        context: RankingContext,
        collection: LiteStoreCollection,
    ): ObjectNode {
        val modelMap = p13nRuntimeUtil.getDVTreatmentSibylModelMappingConfig(context.pageType)
        val modelConfig = modelMap.getOrDefault(collection.rankingType.label, DEFAULT_DV_SIBYL_MODEL_CONFIG)
        val modelName = collectionScorer.getModelName(context, modelConfig)
        val predictorName = collectionScorer.getPredictorName(context, modelConfig)
        return OBJECT_MAPPER_RELAX.valueToTree(mapOf(
            "predictor_name" to predictorName,
            "model_name" to modelName,
        ))
    }
}
```

### New Params Classes for Assembly

```kotlin
/** Vertical: wraps getBoostBundle() + getRankingBundle() + getRankableContent(). */
data class BoostAndRankParams(
    @JsonProperty("boost_by_position_enabled")
    val boostByPositionEnabled: Boolean = false,
    @JsonProperty("deal_carousel_score_multiplier")
    val dealCarouselScoreMultiplier: Double = 1.0,
    @JsonProperty("boost_by_position_allow_list")
    val boostByPositionAllowList: List<String> = emptyList(),
) : StepParams

/** Horizontal: wraps modifyLiteStoreCollection() when(rankingType) dispatch. */
data class RankingSortParams(
    @JsonProperty("ranking_type")
    val rankingType: String,
    @JsonProperty("boost_enabled")
    val boostEnabled: Boolean = false,
    @JsonProperty("business_multipliers")
    val businessMultipliers: Map<String, Double> = emptyMap(),
    @JsonProperty("store_multipliers")
    val storeMultipliers: Map<String, Double> = emptyMap(),
    @JsonProperty("campaign_store_order")
    val campaignStoreOrder: List<String> = emptyList(),
    @JsonProperty("business_sort_overrides")
    val businessSortOverrides: List<String> = emptyList(),
    @JsonProperty("apply_open_closed")
    val applyOpenClosed: Boolean = true,
    @JsonProperty("sort_by")
    val sortBy: String? = null,
    @JsonProperty("always_open_type")
    val alwaysOpenType: String = "CONTROL",
    @JsonProperty("priority_business_ids")
    val priorityBusinessIds: List<String> = emptyList(),
    @JsonProperty("priority_vertical_ids")
    val priorityVerticalIds: List<Long> = emptyList(),
    @JsonProperty("order_history_rerank_enabled")
    val orderHistoryRerankEnabled: Boolean = false,
) : StepParams
```

### AssembledContract

```kotlin
data class AssembledContract(
    val layer: String,                          // "vertical" or "horizontal"
    val contract: UnifiedExperimentConfig,
    val carouselId: String? = null,             // horizontal only
    val rankingType: String? = null,            // horizontal only
    val metadata: Map<String, String> = emptyMap(), // debugging context
)
```

---

## What Gets Logged

Every shadow request emits the assembled contract as a structured log line **plus the full
experimentMap snapshot**. This is the "aha moment" — you can grep for any request and see the
exact contract and all DVs that were active.

### Vertical log example

```
ubp_assembled_contract
  layer=vertical
  request_id=req-abc123
  predictor_name=feed_ranking_fw
  model_name=store_ranker_fw_v3
  blending_config_id=config_v2
  deal_multiplier=1.0
  boost_allowlist_size=3
  nv_unpin_enabled=false
  experiment_map={"enable_homepage_vertical_blender":"enabled","hp_vertical_blending_config":"config_v2","universal_ranker_fw_xp_variant":"treatment_fw_v3",...}
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
          { "id": "score", "type": "MODEL_SCORING", "params": {
              "predictor_ref": "p_act",
              "calibration_config": { "calibration_entries": [...], "default_multiplier": 1.0 },
              "intent_scoring_config": { "feature_configs": [...], "score_lookup_entries": [...], "default_score": 1.0 },
              "vertical_boost_weights": {
                "boosting_multipliers": [{ "vertical_ids": [2], "multiplier": 1.2 }],
                "default_multiplier": 1.0
              },
              "diversity_enabled": false,
              "diversity_scoring_params": { "weight": 0.0 }
            }
          },
          { "id": "boost_and_rank", "type": "BOOST_AND_RANK", "params": {
              "boost_by_position_enabled": true,
              "deal_carousel_score_multiplier": 1.0,
              "boost_by_position_allow_list": ["dt:some_carousel", "nv_carousel"]
            }
          }
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
  boost_enabled=true
  business_overrides_count=2
  campaign_stores_count=0
  always_open_type=SCHEDULE_AHEAD_AFTER_ASAP
  experiment_map={"shouldBoostBusinessForStoreRanker":"enabled","shouldUseSRFWPredictorForStoreRanker":"treatment_fw",...}
  contract={
    "control": {
      "horizontal_pipeline": {
        "steps": [
          { "id": "score", "type": "MODEL_SCORING", "params": { "predictor_name": "feed_ranking_fw", "model_name": "store_ranker_fw_v4" } },
          { "id": "rank", "type": "RANKING_SORT", "params": {
              "ranking_type": "CAROUSEL",
              "boost_enabled": true, "business_multipliers": {"biz_123": 1.3}, "store_multipliers": {},
              "campaign_store_order": [], "business_sort_overrides": ["biz_456", "biz_789"],
              "apply_open_closed": true, "always_open_type": "SCHEDULE_AHEAD_AFTER_ASAP"
            }
          }
        ]
      }
    }
  }

ubp_assembled_contract
  layer=horizontal
  request_id=req-abc123
  carousel_id=mx_logo_1
  ranking_type=MX_LOGO_CAROUSEL
  contract={
    "control": {
      "horizontal_pipeline": {
        "steps": [
          { "id": "score", "type": "MODEL_SCORING", "params": { "predictor_name": "feed_ranking", "model_name": "store_ranker_v1" } },
          { "id": "rank", "type": "RANKING_SORT", "params": {
            "ranking_type": "MX_LOGO_CAROUSEL",
            "apply_open_closed": true,
            "priority_business_ids": ["biz_100", "biz_200"],
            "priority_vertical_ids": [2, 10]
          } }
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
          { "id": "rank", "type": "RANKING_SORT", "params": { "ranking_type": "PICKUP_STORE_FEED", "sort_by": "DISTANCE" } }
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
- Score modifier maps show which businesses/stores get boosted and by how much
- Campaign sort orders show which stores get campaign-positioned
- AlwaysOpen type shows which open/closed sort strategy was used
- The full experimentMap shows exactly which DVs influenced every decision

---

## Vertical Shadow

```kotlin
// In DefaultHomePagePostProcessor.reOrderGlobalEntitiesV2()
// Always runs — no DV gating. Safe because this is sandbox-only code.
val oldResult = rankAndDedupeContent(context, ...)

runCatching {
            // Assemble the contract from what the old path actually used
            val assembled = contractAssembler.assembleVertical(context)

            // Log it — contract + experimentMap snapshot
            logger.info(
                "ubp_assembled_contract layer=vertical " +
                "request_id=${context.requestId} " +
                "predictor_name=${assembled.contract.predictorSummary()} " +
                assembled.metadata.entries.joinToString(" ") { "${it.key}=${it.value}" } + " " +
                "experiment_map=${OBJECT_MAPPER_RELAX.writeValueAsString(context.experimentMap)} " +
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

// Always return old result — shadow never changes user-visible behavior
```

---

## Horizontal Shadow

```kotlin
// In DefaultHomePageStoreRanker.rank()
// Always runs — no DV gating. Safe because this is sandbox-only code.
return scoredCollections.map { collection ->
    CoroutineScope(...).async {
        val oldResult = modifyLiteStoreCollection(context, collection, storesById, metadata, ...)

        runCatching {
                // Assemble per-carousel contract (now includes score modifiers, campaigns, etc.)
                val assembled = contractAssembler.assembleHorizontal(context, collection, metadata)

                logger.info(
                    "ubp_assembled_contract layer=horizontal " +
                    "request_id=${context.requestId} " +
                    "carousel_id=${collection.id} " +
                    "ranking_type=${assembled.rankingType} " +
                    assembled.metadata.entries.joinToString(" ") { "${it.key}=${it.value}" } + " " +
                    "experiment_map=${OBJECT_MAPPER_RELAX.writeValueAsString(context.experimentMap)} " +
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

        oldResult  // always return old result
    }
}.awaitAll()
```

---

## Step Decomposition

We are NOT boiling the ocean. This is step 1. Finer decomposition is a future iteration once
interfaces are proven.

### Vertical (2 steps)

| Step type | What it wraps | Replaces today |
|---|---|---|
| `MODEL_SCORING` | `getScoreBundle()` — Sibyl gRPC + `BlendingUtil.blendBundle()` including diversity rerank (all one atomic call) | `EntityRankerConfiguration.getScoreBundleWithWorkflowHelper()` + `BlendingUtil` |
| `BOOST_AND_RANK` | `getBoostBundle()` + `getRankingBundle()` + `getRankableContent()` — score assignment to domain objects, position boosting, deal multiplier, pin vs flow sort order, reassembly (all one atomic flow) | `Boosting.kt` + `BoostingBundle.boosted()` + `RankingBundle.ranked()` |

### Horizontal (2 steps)

| Step type | What it wraps | Replaces today |
|---|---|---|
| `MODEL_SCORING` | `scoredCollections()` — Sibyl + score modifiers (atomic) | `StoreCollectionScorer` |
| `RANKING_SORT` | `modifyLiteStoreCollection()` `when(rankingType)` dispatch — all ranking-type-specific logic in one step | `DefaultHomePageStoreRanker.modifyLiteStoreCollection()` when-chain |

**Note:** These are intentionally coarse. The old 5-step decomposition (SCORE_MODIFIER,
CAMPAIGN_SORT, BUSINESS_RULES_SORT, ORDER_HISTORY_RERANK) was premature — those are internal
details of `RANKING_SORT`. Finer decomposition happens in a future iteration.

---

## The Convergence Path

```
Phase A: Assembly + Visual Inspection
  Shadow assembles contracts per request, logs them.
  You grep/query the logs, inspect contract JSONs visually.
  Identify: "the shape is stable — vertical always has 2 steps,
  horizontal has 2 default + per-carousel overrides."
  KEY: experimentMap logged alongside → explains every decision.

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

## Critical Rules

1. **New path errors must never affect users.** Wrapped in `runCatching`. On failure: log `ubp_shadow_error`, return old result.
2. **New path latency must not add to p99.** Log duration; alert if exceeds 5ms p99 additional.
3. **Shadow + assembler are temporary.** Delete once static control.json is validated. Don't let them become permanent scaffolding.
4. **Assembled contracts must be valid UBP JSON.** If the assembler produces JSON that the engine can't parse, that's a contract spec bug — fix the spec, not the assembler.
5. **Log experimentMap with every contract.** This is the debug key — without it you can't explain WHY a particular predictor or config was chosen.

---

## Files to Create / Modify

| Action | File |
|---|---|
| New | `libraries/common/.../ubp/shadow/UbpContractAssembler.kt` |
| New | `libraries/common/.../ubp/shadow/AssembledContract.kt` |
| New | `libraries/common/.../ubp/vertical/steps/BoostAndRankStep.kt` (step + params class) |
| New | `libraries/common/.../ubp/horizontal/steps/RankingSortStep.kt` (step + params class) |
| Modify | `DefaultHomePagePostProcessor.kt` — add shadow assembler call after old path |
| Modify | `DefaultHomePageStoreRanker.kt` — add shadow assembler call after old path |

---

## Prerequisites

None. This is the first thing built. The assembler is standalone — it reads existing runtime
utils that already exist. The `FeedRowRanker` reference will be a stub until Parts 2-5 are
complete, but the **assembler can run immediately** and start logging contract JSONs even
before the engine exists. This is the fast path to visual feedback.

---

## Done When

- Assembler runs on sandbox and emits `ubp_assembled_contract` logs for both layers
- **experimentMap snapshot** logged with every contract
- You can grep a `request_id` and see the full vertical + horizontal contract JSONs
- Vertical contracts show: predictor, blending params (including diversity), boost-and-rank params
- Horizontal contracts show: per-carousel step sequences, MODEL_SCORING params, RANKING_SORT params with ranking type mappings
- Contract shape has stabilized across 1000+ requests
- Static `control.json` written from representative assembled contract
- Shadow with static `control.json` shows `divergence_count = 0`
