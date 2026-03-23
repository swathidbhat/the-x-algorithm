# X recommendation algorithm — answers from this repository only

This document answers common questions using **only** the code and documentation in this Git repository (`the-x-algorithm`). It does not use external blogs, papers linked from READMEs, or other out-of-repo sources.

---

## As a content consumer, how does the algorithm decide what to show me?

For the **Home Timeline**, the open-sourced surface is **Home Mixer**, which builds the feed (including **For You**: “best Tweets from people you follow + recommended out-of-network content,” and **Following**: reverse-chronological tweets from people you follow).

The **For You** path described in `home-mixer/README.md` is:

1. **Candidate generation** — Pull candidate tweets from multiple **candidate sources** (examples named in-repo: Earlybird Search Index, User Tweet Entity Graph (UTEG), Cr Mixer, Follow Recommendations Service (FRS)).
2. **Feature hydration** — Load on the order of **~6000 features** used for ranking.
3. **Scoring and ranking** — Score candidates with an **ML model**.
4. **Filters and heuristics** — Apply rules such as author diversity, in-network vs out-of-network balance, feedback fatigue, deduplication / removal of previously seen tweets, and **visibility filtering** (blocked/muted authors, NSFW settings, etc.).
5. **Mixing** — Combine tweets with non-tweet items (ads, Who-to-follow modules, prompts).
6. **Product features** — Conversation modules, social context, pagination/cursoring, etc.

So, at a high level, “what to show” is: **candidates retrieved from several retrieval systems → ML scoring with many features → heuristic and policy filters → mixed with other product modules**.

The **candidate sourcing** stage is documented in `RETREIVAL_SIGNALS.md` as narrowing from a very large corpus to a few thousand items, using **Twitter user behavior** as the main input. Signals listed there include follows/unfollows/mutes/blocks, likes, retweets, quote tweets, replies, shares, bookmarks, clicks, video watch, negative feedback, reports, notification opens, and similar. Those signals are used (per component) as **training labels and/or ML features** in candidate sourcing and light ranking.

For **notifications**, `pushservice/README.md` describes a parallel pattern: eligibility → fetch candidates from multiple sources → hydrate → light filtering → **light then heavy ranking** → heavy filtering → send.

---

## As a content consumer — what is the algorithm prioritising? For chosen content, how is it ranked?

### What the product layer prioritises (besides raw ML scores)

From `home-mixer/README.md`, after scoring, **filters and heuristics** explicitly include:

- **Author diversity**
- **Content balance (in-network vs out-of-network)**
- **Feedback fatigue**
- **Deduplication / previously seen tweet removal**
- **Visibility filtering** (blocked/muted authors/tweets, NSFW settings)

So the feed is not purely “top ML score”; it is **adjusted** for diversity, network mix, fatigue, deduplication, and visibility/safety rules.

The top-level `README.md` states that roughly **~50%** of For You post candidates come from the **search index (in-network)** candidate source, with the rest coming from other sources (out-of-network style retrieval is explicitly part of For You).

### How ranking works (in-repo description)

**Timeline / Home candidate retrieval and light ranking**

- **Earlybird (Tweet Search System)** is used for **in-network** retrieval: given accounts you follow, find their recent tweets and rank them (`src/java/com/twitter/search/README.md`). It uses **Lucene-based** realtime search with incremental indexing and freshness-oriented serving.
- **TimelineRanker** (`timelineranker/README.md`) is a **legacy** service that returns **relevance-scored** tweets from Earlybird and UTEG; it **does not** perform heavy/model-based ranking itself—it uses scores from the Search Index (and UTEG lists). Home Mixer can call Timeline Ranker for Earlybird + UTEG candidates; truncation follows those scores.

**Light ranker (Earlybird)**

`src/python/twitter/deepbird/projects/timelines/scripts/models/earlybird/README.md` describes the **Earlybird light ranker** as a **logistic regression** model predicting the **likelihood the user will engage with a tweet**, with separate configurations for **in-network** (`recap_earlybird`) and **out-of-network / UTEG** (`rectweet_earlybird`). It is positioned as a **lighter** approximation that can score more tweets than the heavy ranker.

Scoring uses **static** tweet features (examples given in-repo: whether it is a retweet, has a link, has trend words at ingestion, is a reply, a **static text quality** score from factors such as offensiveness, content entropy, “shout” score, length, readability) and **realtime** features (e.g. like/reply/retweet counts, and health-model scores such as toxicity/block probability). **Tweepcred** (user reputation) is also referenced as used in scoring.

**Heavy ranker (Home)**

The root `README.md` states that a **heavy ranker** is a **neural network** used to rank candidate posts and is a **main signal** after candidate sourcing. The detailed heavy-ranker documentation referenced from the root README lives **outside** this repository; this repo does not duplicate that model spec.

**Notifications**

`pushservice/src/main/python/models/light_ranking/README.md` describes the notification **light** ranker as pre-selecting candidates before heavy ranking.  
`pushservice/src/main/python/models/heavy_ranking/README.md` describes the notification **heavy** ranker as **multi-task learning** predicting probabilities that the user will **open** and **engage** with the notification.

**Follow / account recommendations (context for “who might appear”)**

`follow-recommendations-service/README.md` explains that the ML ranker returns a score that is a **weighted sum** of **P(follow | recommendation)** and **P(positive engagement | follow)** for ranking account candidates.

---

## As a content creator, how does the algorithm decide to distribute my content — to whom, how widely, and for how long?

This repository **does not** spell out a single “distribution policy” document for creators. What it **does** describe are the **mechanisms** that determine where tweets can surface and over what **horizons** in indexed / graph systems.

### To whom

- **In-network distribution** (people who follow you): **Earlybird** retrieves recent tweets from followed accounts for Home (`src/java/com/twitter/search/README.md`). Distribution to followers is therefore tied to **following relationships** and **in-network retrieval/ranking** from the search index.
- **Out-of-network / discovery-style distribution**: **UTEG** implements collaborative filtering: it uses the viewer’s **weighted follow graph**, traverses user–tweet engagement, and returns top-weighted tweets engaged by others, weighted by how many users engaged and those users’ weights (`src/scala/com/twitter/recos/user_tweet_entity_graph/README.md`). So a creator’s tweet can reach users **who do not follow** the creator if it propagates through **engagements** in that graph.
- **Follow Recommendations Service (FRS)** supplies **FutureGraph** tweets from accounts the viewer may want to follow (`follow-recommendations-service/README.md`) — another path by which a creator’s content might reach non-followers.
- **Real Graph** models **likelihood of interaction between users** (`src/scala/com/twitter/interaction_graph/README.md`); CR-Mixer mentions using signals from stores like **RealGraph** in candidate generation (`cr-mixer/README.md`).
- **SimClusters** embeddings tie users and tweets to **communities**; tweet embeddings update when users favorite tweets, and are used for similarity-style recommendation (`src/scala/com/twitter/simclusters_v2/README.md`).
- **Graph Feature Service** supplies pairwise features such as how many of user A’s followings favorited or follow candidate user C (`graph-feature-service/README.md`) — used as signals in recommendation/ranking stacks, not a full “distribution formula” in-repo.
- **Visibility filtering** can **remove, label, downrank, or otherwise alter** how content is shown depending on safety labels, relationships (block/mute), settings, etc. (`visibilitylib/README.md`).

### How widely

In-repo documentation does **not** give numeric “reach caps” for creators. It does describe **competition and narrowing**: candidate sourcing shrinks roughly **a billion** items to **a few thousand** (`RETREIVAL_SIGNALS.md`). After that, **ranking and heuristics** determine what is actually shown. **Tweepcred** (PageRank-style influence on an interaction graph) feeds into **reputation** used in light ranking and related systems (`src/scala/com/twitter/graph/batch/job/tweepcred/README`, Earlybird README).

### For how long

Concrete **time windows** documented here include:

- **UTEG** (and similar GraphJet user–tweet graphs such as **User Tweet Graph** / **User Video Graph**): in-memory user engagements are kept for roughly **24–48 hours**; older events are dropped (`src/scala/com/twitter/recos/user_tweet_entity_graph/README.md`, sibling READMEs under `src/scala/com/twitter/recos/`).
- **Earlybird** **realtime** index: public tweets from about the **last 7 days** in the realtime cluster; **protected** cluster analogous; **archive** cluster described for older material with specific cutoff wording in `src/java/com/twitter/search/README.md`.

Those windows bound **how long tweets stay in certain indices / graphs**, not a literal “promotion duration” for every surface, which is not fully specified in the docs in this repo.

**Event ingestion for graphs**: **Recos-Injector** streams engagements (favorites, RTs, follows, client events, etc.) into GraphJet-based services (`recos-injector/README.md`), which feeds systems like UTEG.

---

## Therefore, as a content creator, what kind of content should I create?

The repository is **not** a playbook for creators. It **does** describe what kinds of **signals and structures** the systems use, which you can interpret as **what the stack is built to measure and optimize toward** (without claiming any off-repo product strategy):

1. **Engagement prediction** — Light rankers are trained to predict **likelihood of engagement** with a tweet (Earlybird README). Notification models predict **opens** and **engagement** (pushservice heavy ranker README). Multiple signals in `RETREIVAL_SIGNALS.md` are used as **labels** for ML (likes, retweets, replies, clicks, video watch, etc.).
2. **Text and health features** — Earlybird scoring includes **text quality** factors (e.g. offensiveness, entropy, readability) and realtime **health** scores such as toxicity (`src/python/twitter/deepbird/projects/timelines/scripts/models/earlybird/README.md`).
3. **Author reputation** — **Tweepcred** / PageRank-style influence and related mass adjustments appear in reputation and ranking-related docs (`src/scala/com/twitter/graph/batch/job/tweepcred/README`, Earlybird README).
4. **Community / embedding structure** — **SimClusters** updates **tweet embeddings** when users **favorite** tweets; embeddings support similarity and recommendation (`src/scala/com/twitter/simclusters_v2/README.md`).
5. **Safety and visibility** — **Visibility filtering** and **trust and safety models** (root `README.md`, `visibilitylib/README.md`, `trust_and_safety_models/README.md`) can **limit or change** how content is shown.

Anything beyond these documented mechanisms would be speculation and is **not** justified by this repository alone.

---

## Scope note

Some components (for example the **Home heavy ranker** model write-up and **TwHIN**) are referenced from the root `README.md` as living in **other** repositories. This document intentionally summarizes only what is **in this repo**.
