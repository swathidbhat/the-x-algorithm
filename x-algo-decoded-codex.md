# X Algo Decoded (Codex)

## Scope

This note uses only code and in-repo docs from this repository. No outside sources are used.

Where a prose doc and the current implementation differ, I treat the current code as authoritative and call out the difference. Where the repo does not state a creator-facing rule directly, I label the conclusion as an **inference from implementation**.

## 1) As a content consumer, how does the algo decide what to show me?

Short answer: it does not score "all tweets on X." It first retrieves a few thousand plausible candidates for *you* from several systems, hydrates a large feature set for those candidates, scores and reranks them, filters out tweets that should not be shown, and only then mixes the surviving tweets into the final Home timeline.

The high-level repo doc says the For You flow is:

- candidate generation
- feature hydration of roughly 6000 features
- ML scoring and ranking
- filters and heuristics
- mixing with ads and other modules

Evidence: `home-mixer/README.md`, lines 15-43.

The current code path for tweet recommendations is the `ScoredTweetsRecommendationPipelineConfig`, and its active candidate pipelines are:

- static candidates
- cached scored tweets
- in-network Earlybird
- direct UTEG
- Tweet Mixer
- content exploration
- lists
- backfill
- communities

Evidence: `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/ScoredTweetsRecommendationPipelineConfig.scala`, lines 446-460.

Important detail: the README's simplified For You diagram still mentions an FRS candidate pipeline, but the current `candidatePipelines` list above does not include it. The code appears to be the newer truth here.

### What these retrieval paths mean for a viewer

| Retrieval path | What the code uses | What that means for you |
| --- | --- | --- |
| In-network Earlybird | followed user IDs, real-graph/followed-user scores, 24h default or 48h for some user states, with configurable inclusion of replies/retweets | tweets from people you follow and strong in-network authors are a core source of candidates |
| Direct UTEG | your weighted in-network graph, favorite-only social proof, 24h tweet age, minimum favorite social proof | out-of-network tweets liked by people close to your graph can enter your feed |
| Tweet Mixer | many retrieval systems driven by user signals, embeddings, interest models, search history, geo/topic sources, evergreen/video sources | a broad interest-matching layer looks for content beyond your direct follow graph |
| Content exploration | your sub-level categories plus cold-start posts from the last 6 hours | the system can try newer category-matched posts even when they have less history |
| Communities | your community memberships, nullcast non-replies, 48h window | community posts can be pulled specifically because you belong to those communities |
| Cached/backfill/static/lists | prior scoring cache, timeline-service backfill, static candidates, list sources | these are supporting paths that keep the feed populated and stable |

Evidence:

- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/query_transformer/earlybird/EarlybirdInNetworkQueryTransformer.scala`, lines 19-23 and 66-88
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/candidate_pipeline/earlybird/ScoredTweetsEarlybirdInNetworkCandidatePipelineConfig.scala`, lines 53-88
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/query_transformer/UtegQueryTransformer.scala`, lines 18-24 and 33-66
- `src/scala/com/twitter/recos/user_tweet_entity_graph/README.md`, lines 7-13
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/candidate_pipeline/ScoredTweetsDirectUtegCandidatePipelineConfig.scala`, lines 48-83
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/query_transformer/ContentExplorationQueryTransformer.scala`, lines 18-24
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/candidate_source/ContentExplorationCandidateSource.scala`, lines 23-25 and 43-77
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/query_transformer/earlybird/CommunitiesEarlybirdQueryTransformer.scala`, lines 29-47
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/candidate_pipeline/earlybird/ScoredTweetsCommunitiesCandidatePipelineConfig.scala`, lines 47-55
- `tweet-mixer/server/src/main/scala/com/twitter/tweet_mixer/product/home_recommended_tweets/HomeRecommendedTweetsRecommendationPipelineConfig.scala`, lines 271-332

### What stops a candidate from reaching your feed

Even after retrieval, many tweets are removed before final serving. The repo shows filters for:

- age: default 48-hour age filter in the scored-tweets recommendation layer
- self-authored tweets
- retweet deduplication
- location constraints
- previously seen or previously served tweets
- feedback fatigue from "See fewer"
- invalid subscription replies
- low-quality or unsafe content filters after scoring, including slop, gore, NSFW, spam, violent content, language filtering, duplicate conversation tweets, and support-account replies

Evidence:

- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/ScoredTweetsRecommendationPipelineConfig.scala`, lines 471-490 and 532-556
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/functional_component/filter/PreviouslySeenTweetsFilter.scala`, lines 24-45
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/functional_component/filter/FeedbackFatigueFilter.scala`, lines 39-79

### Final consumer-facing implication

As a consumer, the algo is deciding in roughly this order:

1. Which tweets are even plausible for *this viewer*?
2. Which of those look most valuable according to prediction models?
3. Which of those should still be removed for safety, repetition, policy, fatigue, or feed-balance reasons?
4. How should the remaining tweets be mixed with ads and modules into the final timeline?

That last mixing step matters: the final For You mixer does not only return ranked tweets. It also inserts ads, who-to-follow, who-to-subscribe, previews, edited-tweet replacements, and a new-tweets pill.

Evidence: `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/for_you/ForYouScoredTweetsMixerPipelineConfig.scala`, lines 158-174 and 187-297.

## 2) As a content consumer, what is the algo prioritizing to show in my feed? For the content that it decides to show, how does it rank the content?

Short answer: the ranking code is optimizing for tweets that are predicted to drive a bundle of positive actions from you, while avoiding negative feedback, and then reshaping that ranked list so the feed is not too repetitive, too reply-heavy, too unsafe, or too dominated by one source.

### What the model is explicitly optimizing for

The ranking layer has explicit prediction heads for:

- favorites
- replies
- retweets
- replies that get engaged by the author
- good clicks
- good clicks with dwell
- good profile clicks
- video quality views
- immersive video quality views
- bookmarks
- shares
- dwell
- video quality watches
- video watch time
- negative feedback

Evidence: `home-mixer/server/src/main/scala/com/twitter/home_mixer/model/PredictedScoreFeature.scala`, lines 62-311.

That means the code is not optimizing for just one thing like likes. It is explicitly scoring a tweet on many possible outcomes, including negative ones.

### How the score is actually built

The current ranking flow is:

1. Retrieve candidates from the source pipelines.
2. Hydrate a large feature set for those candidates.
3. Run the heavy ranker model(s).
4. Combine the model heads with query-time weights into a final weighted score.
5. Apply request-specific heuristic and listwise rescoring.
6. Sort by final score and then apply feed-shaping selectors.

Evidence:

- `home-mixer/README.md`, lines 17-31
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/scoring_pipeline/ScoredTweetsModelScoringPipelineConfig.scala`, lines 169-312
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/scoring_pipeline/ScoredTweetsRerankingScoringPipelineConfig.scala`, lines 17-39
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/functional_component/scorer/WeighedModelRerankingScorer.scala`, lines 38-69
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/util/RerankerUtil.scala`, lines 18-40 and 91-137
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/ScoredTweetsRecommendationPipelineConfig.scala`, lines 522-577

### What kinds of features it uses to rank

The heavy model scoring pipeline hydrates a very broad feature set, including:

- author features
- earlybird/search features
- graph two-hop and in-network features
- viewer-author and viewer-related-user real-graph features
- SimClusters engagement similarity and user-tweet scores
- topic features
- tweet content and semantic-core features
- TwHIN author and tweet features
- view counts
- UTEG social-proof features
- real-time aggregates
- creator metrics
- large embeddings
- transformer embeddings
- Grok annotations
- media completion
- multimodal embeddings

Evidence: `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/scoring_pipeline/ScoredTweetsModelScoringPipelineConfig.scala`, lines 203-306.

So the feed is not being ranked by a single "engagement count." It is being ranked with a large learned feature space that mixes graph proximity, topical similarity, content semantics, media behavior, recent engagement, and creator-level signals.

### What the feed is prioritizing after raw model scoring

After the model score is built, the code still reshapes the ranking with heuristics and listwise rescoring. The active rescorers include:

- out-of-network rescoring
- reply rescoring
- multi-task normalization
- content-exploration rescoring
- deep-retrieval rescoring
- evergreen deep-retrieval rescoring
- author-based diversity decay
- impressed-author decay
- impressed-media and impressed-image decay
- candidate-source diversity decay
- feedback-fatigue rescoring
- multimodal rescoring
- live-content rescoring

Evidence: `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/scorer/HeuristicScorer.scala`, lines 30-56.

Two especially important feed-shaping mechanisms are:

- repeated tweets from the same author are decayed down-list rather than all receiving equal treatment
- repeated tweets from the same candidate source/reason are also decayed, except for in-network served type

Evidence:

- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/scorer/AuthorBasedListwiseRescoringProvider.scala`, lines 14-54
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/scorer/CandidateSourceDiversityListwiseRescoringProvider.scala`, lines 18-49

### Final ranking constraints before the feed is shown

Even after sorting by score, the final selectors still impose structure:

- communities are capped to 1 tweet per community and 3 community tweets total
- candidates are sorted by `ScoreFeature`
- some content-exploration and deep-retrieval candidates can get fixed-position boosts
- the For You mixer debunches out-of-network content so there are at most 2 consecutive out-of-network candidates

Evidence:

- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/selector/KeepTopKCandidatesPerCommunity.scala`, lines 11-35
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/ScoredTweetsRecommendationPipelineConfig.scala`, lines 558-577
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/for_you/ForYouScoredTweetsMixerPipelineConfig.scala`, lines 125 and 187-209

### Final consumer-facing answer

What the algo appears to prioritize in your feed is:

- tweets it predicts you are most likely to engage with positively across several action types
- tweets with low predicted negative feedback
- tweets that are relevant through your graph, interests, categories, communities, or recent signals
- tweets that keep the feed balanced, rather than overloading you with the same author, same source, or too many out-of-network items in a row

## 3) As a content creator, how does the algorithm decide to distribute my content? Specifically, how does it decide to whom, how widely to distribute, and for how long to distribute?

There is no single file in the repo that says "creator distribution policy." So this section is partly an **inference from implementation**. The code strongly suggests that distribution is emergent from retrieval eligibility, ranking strength, and filtering, not from one monolithic "boost/spread" switch.

### To whom does it distribute a post?

The code suggests four main audience routes:

1. **Followers and near-followers**

In-network Earlybird retrieval uses followed user IDs, viewer state, and real-graph or followed-user scores. That is the clearest direct path for a creator's post to reach people already connected to them.

Evidence:

- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/query_transformer/earlybird/EarlybirdInNetworkQueryTransformer.scala`, lines 68-88
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/candidate_pipeline/earlybird/ScoredTweetsEarlybirdInNetworkCandidatePipelineConfig.scala`, lines 72-88

2. **Out-of-network viewers connected through recent engagers**

UTEG is explicitly used for the "XXX liked" style of out-of-network recommendation. It uses the viewer's weighted follow graph as seeds and returns tweets engaged by those weighted users. In Home, the direct UTEG query uses favorite-only social proof and requires at least one non-author favorite in social proof.

Evidence:

- `src/scala/com/twitter/recos/user_tweet_entity_graph/README.md`, lines 7-13
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/query_transformer/UtegQueryTransformer.scala`, lines 35-63
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/candidate_pipeline/ScoredTweetsDirectUtegCandidatePipelineConfig.scala`, lines 78-83
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/feature_hydrator/UtegFeatureHydrator.scala`, lines 83-98

3. **Interest-matched viewers**

Tweet Mixer retrieves candidates through many audience-matching systems: SimClusters, TwHIN, deep retrieval, search-history signals, popular geo/topic pipelines, evergreen pipelines, content exploration variants, user-interest summary, and video-oriented pipelines.

Evidence:

- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/candidate_pipeline/ScoredTweetsTweetMixerCandidatePipelineConfig.scala`, lines 64-78
- `tweet-mixer/server/src/main/scala/com/twitter/tweet_mixer/product/home_recommended_tweets/HomeRecommendedTweetsRecommendationPipelineConfig.scala`, lines 271-332
- `tweet-mixer/server/src/main/scala/com/twitter/tweet_mixer/candidate_pipeline/DeepRetrievalUserTweetSimilarityCandidatePipelineConfig.scala`, lines 71-107
- `tweet-mixer/server/src/main/scala/com/twitter/tweet_mixer/candidate_pipeline/EvergreenDRUserTweetCandidatePipelineConfig.scala`, lines 58-77
- `tweet-mixer/server/src/main/scala/com/twitter/tweet_mixer/functional_component/transformer/EvergreenVideosQueryTransformer.scala`, lines 15-31

4. **Community/category-specific viewers**

Communities retrieval keys off the viewer's community memberships. Content exploration keys off viewer categories and recent cold-start posts.

Evidence:

- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/query_transformer/earlybird/CommunitiesEarlybirdQueryTransformer.scala`, lines 37-47
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/query_transformer/ContentExplorationQueryTransformer.scala`, lines 18-24
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/candidate_source/ContentExplorationCandidateSource.scala`, lines 43-77

### How widely does it distribute a post?

**Inference from implementation:** width appears to expand when a post can be retrieved by more than one path *and* continues to rank well after scoring and filtering.

The code points to these widening and narrowing forces:

- widening force: multiple retrieval routes can surface the same post to different viewers
- widening force: strong positive predictions across the ranking heads help a post survive selection
- widening force: positive source signals are used inside Tweet Mixer blending, including weighted priority for favorite, bookmark, share, retweet, and reply signals
- narrowing force: source-specific caps and top-k filters limit how many candidates survive per path
- narrowing force: author diversity and source diversity decay repeated exposure
- narrowing force: previously seen/served filters, location/language filters, feedback fatigue, and safety filters can suppress otherwise eligible posts
- narrowing force: some reply structures are filtered before they can compete

Evidence:

- `tweet-mixer/server/src/main/scala/com/twitter/tweet_mixer/feature/USSFeatures.scala`, lines 128-205
- `tweet-mixer/server/src/main/scala/com/twitter/tweet_mixer/functional_component/TweetMixerFunctionalComponents.scala`, lines 160-188 and 270-294
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/filter/UtegMinFavCountFilter.scala`, lines 17-27
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/filter/OONReplyFilter.scala`, lines 16-40
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/filter/QualifiedRepliesFilter.scala`, lines 28-42
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/scorer/AuthorBasedListwiseRescoringProvider.scala`, lines 14-54
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/scorer/CandidateSourceDiversityListwiseRescoringProvider.scala`, lines 18-49
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/ScoredTweetsRecommendationPipelineConfig.scala`, lines 471-490 and 532-556

There is also a viewer-side gate that matters a lot for out-of-network reach: if a viewer disables For You recommendations, out-of-network candidate pipelines are turned off for that viewer.

Evidence: `home-mixer/server/src/main/scala/com/twitter/home_mixer/functional_component/gate/AllowForYouRecommendationsGate.scala`, lines 9-23.

### For how long does it distribute a post?

The repo does **not** implement one universal distribution lifetime. Different retrieval paths use different windows:

- in-network Earlybird uses 24 hours by default and 48 hours for some viewer states
- Home UTEG uses a 24-hour tweet-age window, while the UTEG service doc says it keeps engagement memory for 24-48 hours
- communities retrieval uses a 48-hour since-time filter
- content exploration scans only the last 6 hours of cold-start posts
- Tweet Mixer's USS signal helper can filter old viewer signals using a 24-hour max age
- evergreen deep-retrieval and evergreen-video pipelines show that some content can stay retrievable beyond short recency windows

Evidence:

- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/query_transformer/earlybird/EarlybirdInNetworkQueryTransformer.scala`, lines 20-23 and 70-72
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/query_transformer/UtegQueryTransformer.scala`, lines 22-24 and 39-58
- `src/scala/com/twitter/recos/user_tweet_entity_graph/README.md`, lines 12-13
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/query_transformer/earlybird/CommunitiesEarlybirdQueryTransformer.scala`, lines 29 and 44-47
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/candidate_source/ContentExplorationCandidateSource.scala`, lines 43-52
- `tweet-mixer/server/src/main/scala/com/twitter/tweet_mixer/feature/USSFeatures.scala`, lines 59-61 and 189-205
- `tweet-mixer/server/src/main/scala/com/twitter/tweet_mixer/candidate_pipeline/EvergreenDRUserTweetCandidatePipelineConfig.scala`, lines 54-77
- `tweet-mixer/server/src/main/scala/com/twitter/tweet_mixer/functional_component/transformer/EvergreenVideosQueryTransformer.scala`, lines 15-31

There is also short-term caching of already scored tweets. Cached scored tweets only survive while their `lastScoredTimestampMs` is within the configured TTL, and a nearby comment says recent negative-feedback checks are tied to a 3-minute cache-TTL window.

Evidence:

- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/feature_hydrator/CachedScoredTweetsQueryFeatureHydrator.scala`, lines 44-55
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/side_effect/CachedScoredTweetsSideEffect.scala`, lines 46 and 191-195
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/gate/RecentFeedbackCheckGate.scala`, lines 10-14

### Final creator-facing answer

**Inference from implementation:** the algorithm appears to distribute a creator's content to:

- the creator's followers and graph-near viewers first
- then to out-of-network viewers whose network recently engaged with it
- then to viewers whose interests, embeddings, search history, categories, communities, or media preferences make it a good match

How wide it goes depends on:

- how many retrieval paths can find it
- how strongly it scores on the ranking objectives
- whether it avoids negative feedback and filtering
- whether it survives diversity, deduping, and feed-balance constraints

How long it stays alive depends on which retrieval path is carrying it. Some paths are very fresh, while evergreen paths can keep some content in circulation longer.

## 4) Therefore, as a content creator, what kind of content should I create?

This is necessarily an **inference from implementation**, but the repo gives a surprisingly clear answer.

If you want to make content that this codebase is structurally set up to reward, create content that does as many of these things as possible:

### A. Create content that earns multiple kinds of positive action, not just likes

The ranking layer explicitly predicts favorites, replies, retweets, bookmarks, shares, good clicks, profile clicks, dwell, video quality views/watches, and watch time.

So the strongest creator strategy implied by the code is not "optimize for one vanity metric." It is "create posts that generate several forms of meaningful engagement."

Evidence: `home-mixer/server/src/main/scala/com/twitter/home_mixer/model/PredictedScoreFeature.scala`, lines 62-311.

### B. Create content that is easy for the system to match to an audience

The retrieval stack contains interest/category/embedding/community machinery:

- SimClusters
- TwHIN
- deep retrieval with user embeddings
- user categories
- communities
- search-history and topic/geo pipelines
- semantic and multimodal features in ranking

That suggests content with clear topical identity, audience fit, and semantic coherence has more ways to be retrieved.

Evidence:

- `tweet-mixer/server/src/main/scala/com/twitter/tweet_mixer/product/home_recommended_tweets/HomeRecommendedTweetsRecommendationPipelineConfig.scala`, lines 271-332
- `tweet-mixer/server/src/main/scala/com/twitter/tweet_mixer/candidate_pipeline/DeepRetrievalUserTweetSimilarityCandidatePipelineConfig.scala`, lines 71-107
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/query_transformer/ContentExplorationQueryTransformer.scala`, lines 18-24
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/scoring_pipeline/ScoredTweetsModelScoringPipelineConfig.scala`, lines 203-306

### C. Create content that your intended audience is likely to actively favorite/bookmark/share/reply to

Why this matters in the code:

- UTEG distribution is driven by engagement from the viewer's weighted graph and uses favorite social proof
- Tweet Mixer weighted signal priority gives explicit priority weight to favorite, bookmark, share, retweet, and reply signals

So engagement from the *right users* appears especially valuable, not just raw impressions.

Evidence:

- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/query_transformer/UtegQueryTransformer.scala`, lines 35-63
- `src/scala/com/twitter/recos/user_tweet_entity_graph/README.md`, lines 7-10
- `tweet-mixer/server/src/main/scala/com/twitter/tweet_mixer/feature/USSFeatures.scala`, lines 142-160

### D. Prefer strong standalone posts, and be careful about narrow reply structures

The repo contains explicit reply filters. Recommended replies to not-followed users can be removed, and some in-network replies are removed when they look like retweets, invalid targets, or reply-to-reply / directed-at structures.

That implies a creator should not assume every reply-thread post is equally distributable in Home.

Evidence:

- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/filter/OONReplyFilter.scala`, lines 16-40
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/filter/QualifiedRepliesFilter.scala`, lines 28-42

### E. For video, create videos that hold quality attention

The ranking code has explicit heads for video quality views, video quality watches, and watch time. But there is also a filter that can remove videos below a duration threshold, and another branch in that same filter removes suspicious high-completion long videos.

So the repo points toward video that is long enough to qualify, then strong enough to hold real attention.

Evidence:

- `home-mixer/server/src/main/scala/com/twitter/home_mixer/model/PredictedScoreFeature.scala`, lines 131-190 and 249-290
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/functional_component/filter/MinVideoDurationFilter.scala`, lines 17-60

### F. Avoid content that triggers negative feedback, fatigue, or safety filters

The ranking stack explicitly models negative feedback, and the serving stack filters for slop, gore, NSFW, spam, violent content, language issues, duplicate conversation content, and feedback fatigue from "See fewer."

That means distribution is not only about earning positive engagement. It is also about avoiding the forms of content that get suppressed or filtered.

Evidence:

- `home-mixer/server/src/main/scala/com/twitter/home_mixer/model/PredictedScoreFeature.scala`, lines 260-311
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/ScoredTweetsRecommendationPipelineConfig.scala`, lines 532-556
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/functional_component/filter/FeedbackFatigueFilter.scala`, lines 39-79

### G. Do not rely on flooding the timeline

The ranking heuristics explicitly decay repeated exposure from the same author and repeated exposure from the same source/reason. Communities are also capped. Previously seen/served content is filtered.

So the repo suggests that publishing more is not the same thing as getting proportionally more distribution.

Evidence:

- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/scorer/AuthorBasedListwiseRescoringProvider.scala`, lines 14-54
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/scorer/CandidateSourceDiversityListwiseRescoringProvider.scala`, lines 18-49
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/selector/KeepTopKCandidatesPerCommunity.scala`, lines 14-15
- `home-mixer/server/src/main/scala/com/twitter/home_mixer/functional_component/filter/PreviouslySeenTweetsFilter.scala`, lines 32-45

## Bottom line

If I compress the repo's logic into one creator strategy, it is this:

Create content that a clearly matched audience will actively **favorite, reply to, retweet, bookmark, share, click into, dwell on, and watch**, while minimizing negative feedback and avoiding structures that the Home pipeline filters away.

The codebase looks most favorable to content that is:

- audience-matchable
- engagement-rich across multiple action types
- safe and low-regret
- not overly repetitive
- strong enough to win both retrieval and ranking, not just one of them
