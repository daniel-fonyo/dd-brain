# Overview

_**Ranking**_ in Homepage / Vertical Landing Page / Carousel Landing Page / Pill Filter Landing Page is done in 3 stages:

- **First Pass Ranking** happens at the retrieval stage: Among all the deliverable stores, the ranker selects the set of candidates which are more relevant to the Cx
    
- **Ranking stage (SPR1) (a.k.a. Store Ranker):** happens inside of Feed Service by using the prediction scores (pConv) from a store ranker model in Sibyl to rank each entity within carousels, as well as the first page (up to 50) of store feed
    
- **Post-Processing stage (SPR2)** also happens inside of Feed which ranks all the available carousels and 1st page of the store feed vertically through Universal Ranker from Sibyl, and then applies dedupe
    

|   |   |   |   |   |
|---|---|---|---|---|
||**Homepage**|**Carousel Landing Page**|**Vertical Landing Page**|**Pill Filter Landing Page**|
|First Pass Ranking|Yes|Yes|No|No|
|SPR1|Store Ranker|Store Ranker|LR15|Store Ranker|
|SPR2|Universal Ranker|No|Programmatic Carousel Ranker|No|

## Can we pin carousels to the top of homepage?

We have a pinning allowlist that is maintained by p13n team. [https://github.com/doordash/runtime/blob/master/data/feed-service/carousels/pinned_carousel_ranking_order.json](https://github.com/doordash/runtime/blob/master/data/feed-service/carousels/pinned_carousel_ranking_order.json)

# First Pass Ranker

First Pass Ranker, a.k.a FPR, is a filtering mechanism aimed to ensure recommendation relevance while bounding the number of results returned from candidates retrieval. Currently, a store level FPR is introduced in Search Service to ensure for each Cx, the stores returned from ElasticSearch put higher priorities to relevance.

The above plot demonstrates a POC workflow that will only run for 1 submarket (currently we selected submarket 6). During the offline process, an ETL job will produce a table of consumer id, consumer geo hash and its corresponding store candidates. In the online process the _ElasticSearch_ query will take consideration of the store candidates and only return the valid stores. If Search DB fails, we will fall back to the original flow.

In this POC we define the trigger as follows:

**Trigger:** 

We will only trigger FPR for selected consumers that has ordered at least X number of deliveries from doordash with a combination of strategies in the following priority up to 1300 stores

1. Top Merchant strategy: to include all national fav store
    
2. Reorder Strategy: to include the stores that consumer has order in the past 3 (configurable) month
    
3. X-vertical strategy: to include selected verticals
    
4. Recommendation strategy: we will first try store2vec (see [Algorithms](https://docs.google.com/document/d/1h3Ei9s-n_tjSP9Rj2aTHegUIXO4L4to_MdQZwv36vJQ/edit#heading=h.57cvw8j50e72) section)
    
5. New store strategy: to include stores that l28d order volume is below 10
    

Strategy 1 through 5 will consists on average 900 stores, but capped at 1500 stores.

Currently the FPR strategy covers around 70%+ of consumers, the remaining are consumers who has < X number of deliveries from last 28 days in the certain geolocation, then the selection of top 1500 stores are purely based on distance.

**Fusion:**

The fusion step will combine all store candidates from all the strategies and validate by the search service for radius validation.

  
More information can be found from the [First Pass Ranker Eng RFC](https://docs.google.com/document/d/1h3Ei9s-n_tjSP9Rj2aTHegUIXO4L4to_MdQZwv36vJQ/edit#)

# Store Ranker

pConv (Probability of Conversion) was a machine learning model used to rank stores in Store Ranker (SPR1) stage. Starting from Q2 2022, the team has been experimenting a Pairwise Store Relevance Ranker ([Spec](https://doordash.atlassian.net/wiki/spaces/Eng/pages/2909078001), [RFC](https://docs.google.com/document/d/1zAjVtR_52onYlPo3aSm6ssCokq4H4hTZ474WFIuVH_4/edit)) to better recommend stores based on user engagement signals. Since Q3 2022, carousels and store feed have been ranked by the new Pairwise Store Ranker except vertical landing pages (to test in Q4) which are still ranked by the logistic regression based model served from Sibyl Prediction Service. The high level description of the model can be found [https://docs.google.com/presentation/d/1lLlMC2aZyOk6RCbzk--MDDg6CiC3u4RshEwQvDEV6EY/edit#slide=id.g30c9fa3ce2_0_300](https://docs.google.com/presentation/d/1lLlMC2aZyOk6RCbzk--MDDg6CiC3u4RshEwQvDEV6EY/edit#slide=id.g30c9fa3ce2_0_300) . The list of features being used and it’s corresponding coefficients can be found [here](https://docs.google.com/spreadsheets/d/1GZv6XwfjOxAryunbyfmoeh9sP6m741yHZLl_7GEG_As/edit#gid=1271646108).

**Implementation:**

Inside of the Feed Service Ranking module, Feed will generate a request with the right entity ids (consumer_ids, store_ids) and real time features (store eta, delivery fee) to Sibyl Prediction Service’s GetPrediction endpoint to get the pConv score, with predictor name feed_ranking .

[Code](https://github.com/doordash/feed-service/blob/10342b675ee3fc9a750a12374963f7b8322e7f37/common/src/main/kotlin/com/doordash/consumer/feed/common/discovery2/domains/scoring/StoreCollectionScorer.kt)

## Ranking Rules

As mentioned above, the Store Ranker model will be used to rank 1st page of Store Feed as well as the stores within each carousel. However, there are Ranking Rules that may adjust / override the ranking from the descending order of the predicted pConv scores.

### Store List (a.k.a Store Feed)

Stores within store list will first be ranked according to the following store status priority:

- Delivery available
    
- Pickup only
    
- Closed
    

On the vertical landing page of the cross-vertical homepage world, the stores are ranked based on the following store status and vertical information:

- Delivery available and primary vertical matched
    
- Pickup only and primary vertical matched
    
- Delivery available and secondary vertical matched
    
- Pickup only and secondary vertical matched
    
- Closed and primary vertical matched
    
- Closed and secondary vertical matched
    

For the stores in the same status, they will be then ranked according to the scores computed by the pConv model prediction in a descending order. Ranking type is “store_feed”, which can be mapped to the corresponding model id from [dv_treatment_sibyl_model_mapping](https://github.com/doordash/runtime/blob/master/data/feed-service/ranking/dv_treatment_sibyl_model_mapping.json).

### Programmatic Carousel

Programmatic carousels are the store carousels which programmatically determine the selection of stores based on specific criteria. For example, the current programmatic carousels include:

- “Order Again”: include stores that the consumer has ordered in the past
    
- “Fastest Near You”: include stores whose ETA is under X mins (X is configured in [runtime](https://github.com/doordash/runtime/blob/cb822d57553ad22e416d337154eaef62bfb9d484/data/feed-service/carousels/eta.json#L9))
    
- “[Try Something New](https://docs.google.com/document/d/1y5KVIAFXb6NTZVVNqQVEaKrm7nXFOLQYqBF7R-hzyQU/edit)”: include stores that the consumer has not ordered in the past
    

To note that to be included in any store carousel, a store has to be delivery available and has header image. The ranking is then decided based on the scores computed by the pConv model prediction in a descending order. Ranking type is “carousel”, which can be mapped to the corresponding model id from [dv_treatment_sibyl_model_mapping](https://github.com/doordash/runtime/blob/master/data/feed-service/ranking/dv_treatment_sibyl_model_mapping.json).

### Merchandising Carousel

Campaign carousels are store carousels which are created manually inside of carousel manager, and fetched from Promotion Service’s GetCarouselMetadataForConsumer endpoint.

Again, a store has to be delivery available and has header image to be included in the carousel. The ranking will first be decided by the [sort_order](https://github.com/doordash/services-protobuf/blob/cdacfb958465443edf0582f96b92ea166167259c/protos/promotion/placement.proto#L51) set from carousel manager for the campaign placement. To tie break the sort_order (include the unset ones), we will use the scores computed by the pConv model prediction in a descending order. Ranking type is “carousel”, which can be mapped to the corresponding model id from [dv_treatment_sibyl_model_mapping](https://github.com/doordash/runtime/blob/master/data/feed-service/ranking/dv_treatment_sibyl_model_mapping.json).

### Deal Carousel

Deal carousel (a.k.a offer carousel) is fetched from Promotion Service as well, the stores are filtered from the campaigns being fetched whose placement_type = PLACEMENT_TYPE_FEED. Again, a store has to be delivery available and has header image to be included in the carousel. The ranking however is currently based on a composite score from this formula:

_Blended ranking = ML relevance score + Perceived Incentive Amount / 15.0_

(Perceived Incentive Amount in cents, more details found in [https://docs.google.com/document/d/1zj1f1DJtWfZj1CK0OZrjetDxHd8IawrVyc7ZwMPl45c/edit#bookmark=id.kpb068cu5u2a](https://docs.google.com/document/d/1zj1f1DJtWfZj1CK0OZrjetDxHd8IawrVyc7ZwMPl45c/edit#bookmark=id.kpb068cu5u2a) )

ML relevance score is computed by the pConv model prediction in a descending order. Ranking type is “campaign”. Perceived Incentive Amount is computed via a daily ETL job and stored in FeedDB’s rcopy_fact_store_campaign_details table.

The upcoming iterations of ranking of Deal Carousel is currently owned by the Promotion Platform team now.

### Item Carousel

Item carousels are merchandised carousel (campaign carousel) which is fetched from Promotion Service. The horizontal ranking within the carousel is based on the store scores, and then tie-break by item [sort_order](https://github.com/doordash/services-protobuf/blob/cdacfb958465443edf0582f96b92ea166167259c/protos/promotion/placement.proto#L51) within the same store, then reordered by runtime stack size so that items from higher score stores don't stack together.

Example:

sortedItems : (item1, store1), (item2, store1), (item3, store2), (item4, store2), (item5, store3)

stack size: 1 (allow max 1 item from the same store to stack together)

reorderItems: (item1, store1), (item3, store2), (item5, store3), (item2, store1), (item4, store2)

We will use the scores computed by the pConv model prediction in a descending order for the **stores** within the carousel, as we have not have modeling support for item-level ranking.

For vertical ranking through universal ranker, we are treating item carousel the same as store carousel, and feeding the list of store ids from the carousel into Sibyl for pConv prediction.

# Programmatic Carousel Ranker

Currently we have a formula based programmatic carousel ranker that ranks all store carousels + deal carousel based on the composite score, the composite score is computed as:

w1...wN is the discount factors that we will discount the probability of conversion given the position i of the store in the carousel, vs. if it’s at the 1st position of the carousel, i.e. w1 = 1.

More details on the methodology can be found from [https://docs.google.com/document/d/1S411YXv2D23Km4HYbg_jiTbpPozqXcxsbUP09TDea-4/edit#](https://docs.google.com/document/d/1S411YXv2D23Km4HYbg_jiTbpPozqXcxsbUP09TDea-4/edit#)

To note that carousel ranker is currently implemented inside of the Post Processing module of Feed Service.

[Code](https://github.com/doordash/feed-service/blob/c026f2a03d3f2d7f1c8e069f7cacac84b1700627/common/src/main/kotlin/com/doordash/consumer/feed/common/discovery2/domains/ranking/CarouselRankerConfiguration.kt)

# Universal Ranker

Today the carousel ranker ranks carousels by calculating a probability of conversion for each carousel and then ranking the carousels, however carousels are always ensured to be placed above the home feed. In particular conversions for carousels are treated independently and require calibrated probabilities.

We have implemented a [universal ranker model](https://docs.google.com/document/d/1Czdfrgce-ruolKFVjkBOnNEddKJWa7mGJ7PoUMQeWTU/edit#), a neural network based model which jointly ranks stores, carousels, and collections to ensure the ones that have the highest pConv scores are ranked at the top.

Universal ranker’s inference (prediction) path is also implemented inside of Feed Service’s Post Processing module, with predictor name universal_feed_ranking. The universal ranker is able to rank together:

- individual stores (elements of the store list)
    
- store carousels (whether programmatic, campaign, or deal-based)
    
- store collections
    

Universal ranker’s model is based on a set of features which can be broken down to **FOUR** major categories:

1. Consumer attributes (Cx’s food preference, Cx’s taste embedding, Cx’s embedding based on order history, whether Cx has DashPass, etc.)
    
2. Store attributes (Store taste embedding, store’s orders in the past 28 days, whether store is on DashPass, Mx’s embedding based on copurchase history and item semantics, etc, etc.)
    
3. Cx-Mx interactions (Cx’s views / clicks / searches / orders on the store)
    
4. Real-time features (hour of day, store ETA, store<>cx distance, delivery fee, whether store is showing offer badges, etc.)
    

We have been working to improve universal ranker beyond the initial successful launch.

1. [UR v0.1](https://docs.google.com/document/d/1Czdfrgce-ruolKFVjkBOnNEddKJWa7mGJ7PoUMQeWTU/edit) was the 1st version rolled out in Sep 2021.
    
2. [UR v0.2](https://docs.google.com/document/d/1jboPA5XfsqmkPm0MDU7o_p6jA85S1wU-80Fg5crv2zM/edit) was the 2nd version rolled out in April 2022.
    
3. [UR v3](https://docs.google.com/document/d/1u_oGnweGqMh9vFElEow_rx-c2VYKLj35w4Pg5sWFozE/edit) was the 3rd version rolled out in April 2022.
    
4. As of July 2023, [UR v4](https://docs.google.com/document/d/1OLWMbj0VsSVzA4R5FLFFKKF02wfAuLBX_14QTWaEgZU/edit) was the latest version rolled out with 90% of total traffic.
    

A comprehensive list of features used for different models can be found [here](https://docs.google.com/spreadsheets/d/1_EVQr0zlKCVYMxSTefYbFytz-K8n7eoDXoLo6s_fbqY/edit#gid=1431464064).

## [Exploitation vs Exploration](https://docs.google.com/document/d/19JLVvEdo3kKy9pfef2RFoMFXo2pLgGEEwxpdz9aBa2o/edit#heading=h.damts4i2wsqn)

Universal Ranker, or any ranker based on pConv model only relies on the historical signals on the Cx, i.e. the exploitation. It can lead to a suboptimal state as it would miss the options that Cx has not seen sufficiently before which might lead to a higher engagement, i.e. the exploration part.

To address this, we have experimented a [solution based on Multi-armed Bandit (MAB)](https://docs.google.com/document/d/19JLVvEdo3kKy9pfef2RFoMFXo2pLgGEEwxpdz9aBa2o/edit#), which adds an uncertainty model that capture the exploration adjustment on top of the existing pConv score predicted by exploitation model. The exact formula looks like this:

Where the uncertainty is modeled by the inverse of impression count between Cx and Mx (Nc,m), the less the impression, the higher the uncertainty score is.

The constant C is the hyperparameter we can tune to balance out the weight between exploitation and exploration, currently the value is 0.01 for UR v0.2 and the impression based uncertainty model.

We only apply EnE to UR right now, but we can also experiment EnE to Store Rankers later.

Useful links:

- [Code](https://github.com/doordash/feed-service/blob/8a41b5296c71fa0bc6e3246e0e931b81019f64d8/common/src/main/kotlin/com/doordash/consumer/feed/common/discovery2/domains/scoring/EntityScorer.kt)
    
- [List of features used](https://docs.google.com/spreadsheets/d/1GZv6XwfjOxAryunbyfmoeh9sP6m741yHZLl_7GEG_As/edit#gid=1271646108)
    
- [Latest model code](https://github.com/doordash/sibyl-models/blob/master/training_scripts/consumer/universal_feed_ranking/universal_ranker/universal_ranker.py)
    
- [Step by step model update guide](https://docs.google.com/document/d/1OSGJShHpLxO8T-9Ccb8Lcn2Hq2h7Fju4EbWZ1xCYCvU/edit#)
    

## Boosting

Boosting is a new feature we built as part of Universal Ranker, this is intended to adjust the prediction score to reflect the long term values of the Mx that we believe were not captured entirely from the pConv models. There are two types of boosting we have implemented: Position based and Programmatic. Programmatic is proved to deliver more long term value to our Cx and we’ve decided to roll out this feature, while only keeping Position based support for special events like Valentines’ Day, Mothers' Day, etc.

### Position Based

Boosting should help the universal ranker algorithm understand the true long term value of new content. Over time, relevant boosted content should naturally rise to the top and may not require (as much) boosting to increase visibility. 

However, it is important to note that non-restaurant categories will have different use cases, total volume, and order frequency than restaurant orders, so may need to be considered differently by the algorithm.

Examples:

- Rank the Convenience & Grocery carousel within the **Top 5 position** 
    
- Rank the Mother’s Day carousel within the **Top 1 position** 
    
- Rank the Caviar Curated Guides collection within the **Top 8 position**
    

**Tie Breaker Rules:**

- If multiple units are tied for top positions, the units will be ranked by natural relevance based on the universal ranker algorithm
    
- Examples:
    
    - 3 carousels / collections are boosted to the **Top 2** positions → the carousel or collection with the lowest relevance will be pushed to position 3 (the specific carousel/collection varies per user)
        
    - 5 carousels / collections are boosted to the **Top 3** positions → the carousel or collection(s) with the lowest relevance will be pushed to positions 4 & 5 (the specific carousel/ collection varies per user)
        

**Implementation:**

To implement this, we are giving highest priority to the boosting entities set by the operators such that we can accommodate as many requirements as we can. To illustrate this, say we have 2 entities boosted for Top 3 (K=3) and 1 for Top 6 (K=6) positions, the boosting system will make sure 2 of top 3 positions were reserved for the 2 K=3 entities and another position in top 6 is reserved for the 1 K=6 entity.

The exact algorithm works like this:

We start from the lowest K, i.e. 3, and rank the 2 boosted entities + highest scored unboosted. This gives us the top 3 positions in the final ranking, then we move on to K=6 and rank the single boosted entity plus the remaining next 2 highest unboosted and append to the previous 3, this give us the top 6 final ranking. Finally we append all/any remaining flowing entities.To note that flowing entities should be the union of the unboosted entities and the lower boosted entities. A key element here is that we are now able to "reserve" positions for higher boosted positions and avoid inserting flowing entities which will end up pushing lower boosted positions further than they should be.

[Code](https://github.com/doordash/feed-service/blob/master/common/src/main/kotlin/com/doordash/consumer/feed/common/discovery2/domains/ranking/Boosting.kt)

**Boosted Entities:**

Store carousel is enabled with boosting by default. Item carousel has a [feature flag](https://github.com/doordash/runtime/blob/master/data/feed-service/dynamicvalue/enable_item_carousel_boosting.json) for controlling. Store / Item collection also has a [feature flag](https://github.com/doordash/runtime/blob/master/data/feed-service/dynamicvalue/enable_collection_boosting.json) for controlling.

To enable a carousel for boosting, choose manual carousel sort order with a pre-defined position in [Carousel Manager](https://admin-gateway.doordash.com/tools/carousel_manager/edit/7e08bd13-fb42-4c69-979a-fc7a9e4ece2c). And then add carousel id to this [runtime](https://github.com/doordash/runtime/blob/master/data/feed-service/boost_by_position_carousel_id_allow_list.json) as well to enable boosting.

To note that we also has a [global holdout experiment](https://github.com/doordash/runtime/blob/master/data/feed-service/dynamicvalue/enable_boosting.json), where the control group will not have boosting on any of the entity.

### Programmatic Boosting By Future Value Model (FVM)

Position boosting is certainly by heuristic and not scientific. To replace, we have adopted a Programmatic Boosting method which is based on Future Value Model (FVM).

New Categories Analytics team ran an [offline analysis](https://docs.google.com/document/d/1MDAlYlO-f1o_wbprMH-x4sbLeUoAQMYiTu84ZG2MLiA/edit#) to indicate correlation between first order from a new vertical and incremental overall GMV over 3 months. 

For instance, based on the analysis we know that when a Cx places their first 3P Convenience order, they drive an incremental $62 in GMV over 3 months versus those who do not.  The 'boosted' score for 3P Convenience stores will look like: 

where $447 is average GMV per Cx over 3 months