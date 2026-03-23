# X Algorithm Decoded

A deep-dive analysis of X's recommendation algorithm, based entirely on the open-sourced codebase at [github.com/twitter/the-algorithm](https://github.com/twitter/the-algorithm).

---

## Table of Contents

1. [How the Algorithm Decides What to Show You](#1-as-a-content-consumer-how-does-the-algorithm-decide-what-to-show-me)
2. [What the Algorithm Prioritises and How It Ranks](#2-as-a-content-consumer-what-is-the-algorithm-prioritising-and-how-does-it-rank-content)
3. [How the Algorithm Distributes Creator Content](#3-as-a-content-creator-how-does-the-algorithm-distribute-my-content)
4. [What Kind of Content Should Creators Make](#4-as-a-content-creator-what-kind-of-content-should-i-create)

---

## 1. As a Content Consumer, How Does the Algorithm Decide What to Show Me?

The For You timeline is assembled through a **multi-stage pipeline** orchestrated by the `home-mixer` service (built on the `product-mixer` framework). The pipeline has four major phases: **candidate sourcing, feature hydration, scoring/ranking, and filtering/mixing**.

### Phase 1: Candidate Sourcing — Where Do the Tweets Come From?

The algorithm pulls tweet candidates from **multiple independent sources** in parallel, each returning a pool of potential tweets:

| Source | What It Fetches | Default Fetch Limit |
|--------|----------------|-------------------|
| **In-Network (Earlybird)** | Recent tweets from accounts you follow, ranked by a light TensorFlow model in the search index | 600 tweets |
| **UTEG (User-Tweet-Entity-Graph)** | Tweets that people you follow have liked/engaged with (out-of-network social proof) | 300 tweets |
| **TweetMixer** | Blended out-of-network recommendations from multiple underlying ML systems (SimClusters, TwHIN embeddings) | 400 tweets |
| **FRS (Follow Recommendation Service)** | Tweets from accounts the system thinks you should follow | 100 tweets |
| **Communities** | Tweets from X Communities you belong to | 100 tweets |
| **Lists** | Tweets from lists you subscribe to | 100 tweets |
| **Content Exploration** | Discovery/exploration candidates for broadening interests | Configurable |
| **Backfill** | Older cached tweets to fill gaps when fresh candidates are scarce | 200 tweets |

> **Source code**: `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/candidate_pipeline/` contains all candidate pipeline configs. Fetch limits are defined in `ScoredTweetsParam.scala` under `FetchParams`.

The **~50/50 split** is notable: according to the README, roughly 50% of For You tweets come from the **search index (in-network)** and the other ~50% from **out-of-network sources** (UTEG, TweetMixer, FRS, etc.).

### Phase 2: Feature Hydration — What Does the Algorithm Know About Each Tweet?

Before scoring, the system enriches each candidate tweet with **~6,000 features** across many dimensions:

- **Author features**: Follower count, verification status, account age, reputation score (TweepCred/PageRank), safety labels
- **Engagement features**: Favourite count, retweet count, reply count, video view count (real-time and historical aggregates)
- **Content features**: Media type (image/video/link), language, text embeddings, topic annotations, hashtags, mentions
- **Graph features**: RealGraph score (how likely you are to interact with this author), mutual follow status, how many of your following liked the author's posts
- **Embedding similarity**: SimClusters dot product (user-interest vs tweet-cluster alignment, 145K clusters), TwHIN dense embeddings, CLIP multimodal embeddings
- **Safety features**: NSFW score, spam score, toxicity score, abuse labels (from Grok and trust-and-safety models)
- **User history**: Your recent engagement patterns, impression history, feedback fatigue signals

> **Source code**: Feature hydrators live in `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/feature_hydrator/`. The feature definitions are in `home-mixer/server/src/main/scala/com/twitter/home_mixer/model/HomeFeatures.scala`.

### Phase 3: Scoring — The Heavy Ranker

Each candidate is scored by a **neural network model** (the "Heavy Ranker") served via **Navi** (a Rust-based ML serving framework). The model predicts the probability of 17 different engagement types and combines them into a single score.

The scoring formula from `NaviModelScorer.scala`:

```
weightedScore = SUM( predicted_probability[engagement_type] * weight[engagement_type] )

if weightedScore > 0:
    finalScore = weightedScore + epsilon    (epsilon = 0.001)
if weightedScore < 0:
    finalScore = (weightedScore + negativeWeightsSum) / totalWeights * epsilon
```

### Phase 4: Filtering and Mixing

After scoring, tweets pass through **25+ filters** before reaching your feed:

- **Visibility filters**: Blocked/muted authors, legal compliance
- **Safety filters**: NSFW, gore, spam, violent content (via Grok labels)
- **Quality filters**: Minimum engagement thresholds (e.g., UTEG tweets need minimum favourites)
- **Deduplication**: Quote dedup, retweet dedup, media cluster dedup, conversation dedup
- **Age filters**: Tweets older than a threshold are removed
- **Language filters**: Optional filtering of non-native-language tweets
- **Impression filters**: Tweets you've already seen recently are excluded (via bloom filter)

Finally, the surviving tweets are **mixed** with ads, Who-to-Follow modules, Community suggestions, and other non-tweet content before serving.

> **Source code**: Filters are in `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/filter/`. The mixing pipeline is in `ForYouMixerPipelineConfig.scala`.

---

## 2. As a Content Consumer — What Is the Algorithm Prioritising and How Does It Rank Content?

### The 17 Engagement Predictions

The Heavy Ranker (neural network) predicts the probability that **you specifically** will perform each of these actions on a given tweet. Each prediction is multiplied by a configurable weight to form the final score.

**Positive Engagements (boosted):**

| Prediction | What It Measures | Weight Range |
|-----------|-----------------|-------------|
| `PredictedFavoriteScore` | Will you like this tweet? | -10,000 to +10,000 |
| `PredictedRetweetScore` | Will you retweet it? | -10,000 to +10,000 |
| `PredictedReplyScore` | Will you reply? | -10,000 to +10,000 |
| `PredictedReplyEngagedByAuthorScore` | Will the author reply back to your reply? | -10,000 to +10,000 |
| `PredictedGoodClickScore` | Will you click and then favourite/reply? (meaningful click) | -10,000 to +10,000 |
| `PredictedGoodClickV2Score` | Advanced version of meaningful click | -10,000 to +10,000 |
| `PredictedGoodProfileClickScore` | Will you click the author's profile and engage with it? | -10,000 to +10,000 |
| `PredictedVideoPlayback50Score` | Will you watch >50% of the video? | -10,000 to +10,000 |
| `PredictedTweetDetailDwellScore` | Will you dwell on the tweet for 15+ seconds? | -10,000 to +10,000 |
| `PredictedProfileDwelledScore` | Will you dwell on the author's profile for 20+ seconds? | -10,000 to +10,000 |
| `PredictedBookmarkScore` | Will you bookmark it? | -10,000 to +10,000 |
| `PredictedShareScore` | Will you share it? | -10,000 to +10,000 |
| `PredictedShareMenuClickScore` | Will you click the share menu? | -10,000 to +10,000 |

**Negative Engagements (penalised):**

| Prediction | What It Measures | Weight Range |
|-----------|-----------------|-------------|
| `PredictedNegativeFeedbackV2Score` | Will you hit "Not interested"? | -10,000 to +10,000 |
| `PredictedReportedScore` | Will you report this tweet? | -20,000 to 0 |
| `PredictedStrongNegativeFeedbackScore` | Strong dislike signal | -1,000 to 0 |
| `PredictedWeakNegativeFeedbackScore` | Weak dislike signal | -1,000 to 0 |

> **Source code**: All 17 features are defined in `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/scorer/PredictedScoreFeature.scala`. Model weights are in `HomeGlobalParams.scala` under `ModelWeights`.

**Key insight**: The actual weight values default to `0.0` in the open-sourced code and are configured at runtime via feature switches. This means X can dynamically tune which engagements matter most. The Report weight is capped at a more negative range (-20,000 to 0) than other negative signals, indicating reports are the strongest negative signal.

### Post-ML Heuristic Rescoring

After the neural network scores each tweet, the `HeuristicScorer` applies a series of **multiplicative rescoring factors** (applied sequentially — each multiplier compounds):

#### a) Out-of-Network Penalty
```
OutOfNetworkScaleFactor = 0.75 (default)
```
Tweets from accounts you don't follow get their score multiplied by **0.75x**, giving in-network tweets a structural advantage.

#### b) Reply Penalty
```
ReplyScaleFactor = 0.75 (default)
```
Reply tweets get their score multiplied by **0.75x**, meaning standalone original tweets are prioritised over replies.

#### c) MTL Normalization
Multi-task learning normalization adjusts scores based on the author's follower count and whether the tweet is a reply or retweet:
```
alpha = 100.0 / 100.0 = 1.0
beta = 100,000,000
gamma = 5,000,000
```
This normalises scores to prevent authors with massive followings from dominating purely through scale.

#### d) Author Diversity Decay
The algorithm penalises seeing multiple tweets from the same author in a session:
```
rescoringFactor = (1 - floor) * decayFactor^position + floor

AuthorDiversityDecayFactor = 0.5
AuthorDiversityFloor = 0.25
```
So for a repeated author: 1st tweet = 1.0x, 2nd = 0.625x, 3rd = 0.4375x, ... down to a floor of 0.25x. This ensures feed diversity.

#### e) Candidate Source Diversity Decay
```
CandidateSourceDiversityDecayFactor = 0.9
CandidateSourceDiversityFloor = 0.8
```
Prevents any single candidate source (e.g., all from UTEG) from dominating the feed.

#### f) Content Similarity Decay
- **Media cluster decay**: Penalises showing visually similar media repeatedly
- **Image cluster decay**: Penalises repeated similar images (via CLIP embeddings)
- **Impressed author decay**: Penalises authors you've seen recently

#### g) Feedback Fatigue
If you previously clicked "See Fewer" on an author or a type of content, the `FeedbackFatigueScorer` reduces scores from that author/topic.

#### h) Grok Slop Scoring
A Grok-based content quality scorer (`GrokSlopScoreRescorer`) can downrank low-quality or "slop" content.

#### i) Live Content Boost
```
LiveContentScaleFactor = 1.0 (default, configurable up to 10,000x)
```
Live Spaces from verified authors with 1M+ followers can receive a boost.

#### j) Control AI ("Show More/Less")
When users use the "Show more like this" / "Show less like this" controls:
```
ShowLessScaleFactor = 0.05   (reduces score to 5%)
ShowMoreScaleFactor = 20.0   (20x boost)
EmbeddingSimilarityThreshold = 0.67
```
Content similar to what you said "show less" gets crushed to 5% of its score; "show more" gets a 20x boost.

> **Source code**: `HeuristicScorer.scala`, `ScoredTweetsParam.scala` lines 423-800+.

### Safety and Visibility Downranking

The `visibilitylib` applies rule-based downranking for:
- High toxicity scores (tiered: Abusive Quality, Low Quality, High Quality thresholds)
- High spam scores
- Cryptospam
- Untrusted URLs
- Spam replies

These rules can move content to low-quality conversation sections or reduce its ranking entirely.

> **Source code**: `visibilitylib/src/main/scala/com/twitter/visibility/rules/DownrankingRules.scala`

---

## 3. As a Content Creator, How Does the Algorithm Distribute My Content?

Distribution happens through three distinct mechanisms: **in-network delivery, out-of-network recommendation, and push notifications**.

### 3.1 To Whom Is Your Content Distributed?

#### Tier 1: Your Followers (In-Network)

Your tweets automatically enter the candidate pool for everyone who follows you via the **Earlybird search index**. The light ranker (a TensorFlow model running inside Earlybird) provides an initial relevance score using:
- Text relevance signals (BM25 text matching)
- Your reputation score (`USER_REP`, computed via TweepCred/PageRank)
- Engagement counts (favourites, replies, retweets, video views)
- Content type signals (has media, has link, has hashtag, etc.)
- Safety labels (abuse, NSFW, spam flags)

Not all your followers will see your tweet — it competes with up to 600 other in-network candidates in each follower's feed. The Heavy Ranker then determines final position.

#### Tier 2: Beyond Your Followers (Out-of-Network)

Your tweet can reach non-followers through several paths:

**a) UTEG (Social Proof Path)**
When your followers like/retweet/reply to your tweet, it becomes a candidate for *their* followers' feeds via the User-Tweet-Entity-Graph. This is the "X liked" or "X retweeted" distribution mechanism.
- Requires minimum engagement (configurable via `UtegMinFavCountFilter`)
- The `InNetworkFilter` ensures it only surfaces to users who don't already follow you
- RealGraph scores determine which of your engagers' followers see it (weighted by social graph strength)

**b) SimClusters (Interest-Based Path)**
Your tweet gets an embedding in SimClusters' 145K interest clusters. If your tweet's embedding is similar to a user's interest embedding (dot product similarity), it becomes a candidate — even with zero social proof.
- Threshold: Only tweets with 9+ favourites are hydrated for SimClusters scoring (`MinFavToHydrate`)
- Uses approximate cosine similarity for efficiency (6% of full computation)
- Generates ~15,000 candidates per request (6 source tweets x 25 clusters x 100 tweets)

**c) TweetMixer (Blended Recommendations)**
This coordination layer blends candidates from multiple recommendation engines and delivers them to the For You timeline. It can source from SimClusters, TwHIN embeddings, Earlybird search, topic recommendations, and trending content.

**d) FRS (Follow Recommendation Path)**
If the Follow Recommendation Service recommends your account to a user, your recent tweets become candidates in their feed even before they follow you.

#### Tier 3: Push Notifications

The `pushservice` has its own independent pipeline for recommending notifications:
1. Candidate generation from ContentRecommender, Earlybird, FRS, Trends
2. Light filtering and light ranking via ML model
3. Heavy ranking via `PushMLModelScorer` (multi-task learning for open + engagement probability)
4. Quality predicates filter out low-quality candidates
5. Delivered via IBIS notification service

Quality thresholds for notifications are **stricter** than for the timeline — health/quality models (`BqmlQualityModelPredicates`, `BqmlHealthModelPredicates`) must be passed.

### 3.2 How Widely Is Your Content Distributed?

Distribution breadth is determined by:

1. **Follower base**: All followers have your tweet as a candidate (but not guaranteed to see it)
2. **Early engagement velocity**: UTEG social proof requires real engagement from followers — more likes/retweets = wider UTEG distribution
3. **SimClusters relevance**: Strong topic alignment means distribution to more interest-matched users
4. **Quality/Safety scores**: Low safety scores or high toxicity can reduce or eliminate distribution entirely:
   - Visibility rules can **Drop** (completely remove), **Downrank** (reduce visibility), or add **Interstitials** (warning screens)
   - Different product surfaces have different `SafetyLevel` thresholds — notifications (`TimelineHomePromoted`) are stricter than the main timeline (`TimelineHome`)
5. **Out-of-Network scaling**: Even when your tweet reaches non-followers, it faces a **0.75x score penalty** compared to in-network tweets
6. **Creator multipliers**: Configurable per-creator scaling factors exist:
   - `CreatorInNetworkMultiplierParam` (default: 1.0, max: 100.0)
   - `CreatorOutOfNetworkMultiplierParam` (default: 1.0, max: 100.0)

### 3.3 For How Long Is Your Content Distributed?

The code reveals several temporal mechanisms:

1. **Earlybird recency**: The search index naturally favours recent content. The light ranker model (`timelines_recap_replica`) incorporates tweet age as a ranking signal.
2. **Age filters**: `CustomSnowflakeIdAgeFilter` removes tweets older than a configurable threshold
3. **Cache TTL**: Scored tweets are cached for **3 minutes** by default (`CachedScoredTweets.TTLParam`), with a minimum of 30 cached tweets
4. **Impression bloom filter**: Once a user has seen your tweet, it's recorded and filtered from future requests
5. **Backfill pipeline**: Older tweets can be resurfaced via the backfill candidate source (up to 200 tweets), extending the life of high-quality content
6. **Evergreen deep retrieval**: A dedicated pipeline (`EvergreenDeepRetrievalListwiseRescoringProvider`) specifically boosts "evergreen" content — high-quality tweets that remain relevant over time

> **Source code**: Distribution logic spans `cr-mixer/`, `home-mixer/`, `pushservice/`, `simclusters-ann/`, and `timelineranker/`.

---

## 4. As a Content Creator, What Kind of Content Should I Create?

Based purely on what the code reveals about how the algorithm scores, ranks, and distributes content, here are the evidence-backed strategies:

### 4.1 Optimise for the Engagement Predictions That Matter Most

The Heavy Ranker predicts **17 engagement types**. Your content should be designed to trigger the positive ones and avoid the negative ones.

**Highest-value engagements** (based on weight ranges and structural priority in the code):

| Engagement | Why It Matters (Code Evidence) |
|-----------|------------------------------|
| **Likes (Favourites)** | Primary positive signal. Also the gateway to UTEG distribution — your tweet needs likes from followers to reach non-followers. SimClusters requires 9+ favs to even hydrate embeddings (`MinFavToHydrate`). |
| **Bookmarks** | Dedicated prediction feature with full weight range. A high-intent "save for later" signal. |
| **Shares** | Two separate predictions (share + share menu click), indicating the algorithm places high value on share intent. |
| **Extended dwell time** | `PredictedTweetDetailDwellScore` measures 15+ second dwell; `PredictedProfileDwelledScore` measures 20+ second profile dwell. Longer-form, thought-provoking content that holds attention is valued. |
| **Meaningful clicks** | `PredictedGoodClickScore` tracks clicks that lead to further engagement (favourite or reply after clicking). Clickbait that doesn't convert to engagement is not rewarded. |
| **Video watch-through** | `PredictedVideoPlayback50Score` specifically tracks >50% video completion. Shorter, compelling videos that people actually watch are rewarded over long videos that get skipped. |
| **Replies (especially author-engaged)** | `PredictedReplyEngagedByAuthorScore` specifically predicts whether the author will reply back. The algorithm values two-way conversation. |

**Engagements to avoid triggering:**

| Negative Signal | Impact |
|----------------|--------|
| **Reports** | Heaviest negative weight (range: -20,000 to 0). Even a small report probability tanks your score. |
| **"Not Interested" feedback** | `NegativeFeedbackV2` is a strong ranking penalty. |
| **Strong/Weak negative feedback** | Ranges from -1,000 to 0. |

### 4.2 Structural Content Advantages in the Code

**Original tweets > Replies**
Replies face a **0.75x scaling penalty** (`ReplyScaleFactorParam = 0.75`). Post your key thoughts as original tweets, not replies in threads (unless the thread drives genuine conversation).

**In-network engagement is your launching pad**
Out-of-network tweets face a **0.75x penalty** (`OutOfNetworkScaleFactorParam = 0.75`). Your content must first perform well among followers before the algorithm pushes it wider. Build a genuinely engaged follower base.

**Engagement velocity matters for distribution breadth**
- UTEG requires minimum favourites before distributing out-of-network
- SimClusters only hydrates tweet embeddings after 9+ likes
- This means early engagement is critical — if your tweet doesn't get initial traction from followers, it likely won't reach the algorithmic distribution pipelines

### 4.3 Content Diversity and Anti-Spam

**Don't flood the feed**
Author diversity decay means your 2nd tweet shown to a user scores **0.625x**, your 3rd scores **0.4375x**, down to a floor of **0.25x** (`AuthorDiversityDecayFactor = 0.5, Floor = 0.25`). Fewer, higher-quality tweets outperform a high volume of mediocre ones.

**Vary your content type**
Media cluster decay and image cluster decay penalise visually repetitive content. If you keep posting similar-looking images or videos, the algorithm will downrank subsequent ones.

**Avoid content that looks like spam**
The trust-and-safety pipeline (`trust_and_safety_models/`) runs NSFW, abuse, and spam detection. High spam scores trigger downranking rules in `visibilitylib`. Cryptospam and untrusted URLs are specifically targeted.

### 4.4 Leverage the Algorithm's Distribution Paths

**Topic alignment (SimClusters)**
Your content is embedded into 145K interest clusters. Tweets that clearly align with specific topic clusters are more likely to be recommended to interest-matched users via SimClusters. Be clear and specific about your topic — ambiguous content is harder to cluster.

**Social proof (UTEG)**
When your followers like your tweet, it becomes visible to their followers. This is the primary organic growth mechanism. Content that your existing followers genuinely want to like and share creates a cascading distribution effect through the social graph.

**Profile engagement**
`PredictedGoodProfileClickScore` and `PredictedProfileDwelledScore` mean the algorithm values tweets that make people visit and spend time on your profile. Having a compelling, complete profile with a clear identity supports this.

### 4.5 Video-Specific Guidance

- The algorithm specifically tracks **>50% video playback** (`PredictedVideoPlayback50Score`), plus additional video quality view metrics (`VideoQualityViewParam`, `VideoQualityViewImmersiveParam`)
- There are dedicated candidate sources for popular videos (`ScoredTweetsPopularVideosCandidatePipelineConfig`)
- Video tweet carousels have their own injection logic in the For You feed (`ForYouParam`)
- **Implication**: Make videos that people watch at least halfway through. Front-load the compelling content. Shorter videos that get completed likely outperform long videos that get abandoned.

### 4.6 The "Show More/Less" Amplifier

Users can use Control AI to signal preferences:
- "Show more like this" = **20x boost** to similar content (`ControlAiShowMoreScaleFactorParam = 20.0`)
- "Show less like this" = **0.05x penalty** (reduces to 5%, `ControlAiShowLessScaleFactorParam = 0.05`)
- Similarity threshold: 0.67 embedding similarity

Content that inspires "show more" from even a few users gets a massive amplification. Content that triggers "show less" is effectively buried. Create content people would actively want more of.

### 4.7 Conversation and Community

**Engage with replies to your own tweets**
`PredictedReplyEngagedByAuthorScore` is a dedicated positive signal. The algorithm specifically predicts and rewards creators who respond to their audience.

**Communities pipeline**
There is a dedicated `ScoredTweetsCommunitiesCandidatePipelineConfig` that sources tweets from Communities. Being active in relevant Communities creates an additional distribution channel beyond the main feed.

---

## Summary: The Algorithm in One Paragraph

Your For You feed is built by pulling ~1,500+ candidate tweets from 8+ sources (followers, social proof graph, interest clusters, recommendations, communities, lists, exploration, backfill), enriching each with ~6,000 features, scoring them with a neural network that predicts 17 engagement probabilities, applying multiplicative heuristic rescoring (out-of-network penalty, reply penalty, author diversity decay, safety filters, feedback fatigue, content similarity decay), filtering through 25+ quality and safety gates, and mixing with ads and non-tweet modules. As a creator, your content enters this system first through your followers (in-network), and can escape to the wider platform (out-of-network) through social proof (UTEG likes), interest matching (SimClusters), and recommendation services — but only if it earns genuine engagement, avoids negative signals, and passes safety thresholds.

---

*All references are to code and documentation within the `twitter/the-algorithm` repository. No external sources were consulted.*
