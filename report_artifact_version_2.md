# xbot (@TitanTechIn): Project Analysis and Algorithmic Audit Report

*Cross-referencing the xbot codebase and live production metrics with X's open-source recommendation algorithm.*

| | |
|---|---|
| **Target account** | [@TitanTechIn](https://x.com/titantechin) (TitanTech Investments) |
| **Workspace** | `file:///Users/ninja/Downloads/xbot` |
| **Reference repositories** | [twitter/the-algorithm](https://github.com/twitter/the-algorithm) (March 2023 release, the audit's framework) and [xai-org/x-algorithm](https://github.com/xai-org/x-algorithm) (the live system since January 2026, used to check currency) |
| **Audit date** | October 20, 2026 |
| **Status** | Feature-complete, production deployed (v0.1). No post since October 15, 2026. |

> **Currency notice.** The audit's quantitative framework (ranker weights, SimClusters, TweepCred) comes from `twitter/the-algorithm`, released in March 2023. Its weights are a snapshot dated April 5, 2023. Since January 2026, X publishes a newer Grok-based system at `xai-org/x-algorithm`, and its live weight defaults differ materially: reply is 5.0 instead of 13.5, and there is no direct counterpart to the +75.0 author-reply term. Sections 3.1 to 3.6 therefore show both generations. The direction of the findings holds in both; the exact multipliers should not be used as current targets.

**Evidence labels used in this report**

- **Observed**: measured on the live profile or in the repository.
- **Verified**: confirmed in X's published code, with the file named.
- **Plausible**: consistent with the published design but not directly confirmed.
- **Hypothesis**: a mechanism that cannot be proven from outside the platform and needs an experiment.

---

## Contents

- [Executive Summary](#executive-summary)
- [1. System Architecture and Content Pipeline](#1-system-architecture-and-content-pipeline)
- [2. Quantitative Results and Live Production Audit](#2-quantitative-results-and-live-production-audit)
- [3. Reverse-Engineering X's Algorithm](#3-reverse-engineering-xs-algorithm)
- [4. Technical System Issues](#4-technical-system-issues)
- [5. The Algorithmic Engineering Blueprint](#5-the-algorithmic-engineering-blueprint)
- [6. Code Modifications](#6-code-modifications)
- [7. Roadmap and Priority Matrix](#7-roadmap-and-priority-matrix)
- [Appendix A: Evidence and Confidence](#appendix-a-evidence-and-confidence)
- [Appendix B: Sources](#appendix-b-sources)
- [Appendix C: Glossary](#appendix-c-glossary)

---

## Executive Summary

The **xbot** project is an autonomous cryptocurrency market-intelligence engine that operates the [@TitanTechIn](https://x.com/titantechin) account on X. The engineering is disciplined: 184 passing unit tests, live Binance futures WebSocket streams, sub-second trigger detection, and custom 1600 × 1600 OKX-inspired infographics.

The account, however, has a severe distribution problem, which this report calls **the 9,125-post / 30-view paradox**. Despite 9,125 lifetime posts, recent posts stall at 20 to 50 views with no likes and no replies. The five posts examined in Section 2 drew 27 to 33 views (mean 30.4).

![Figure 1. Views, likes and replies on the five audited posts, with the account's headline numbers](images/fig01_reach_paradox.png)

*Figure 1. Views, likes and replies on the five audited posts, with the account's headline numbers.*

Cross-referencing the codebase with X's published ranking code points to five findings:

| # | Finding | Evidence status |
|---|---|---|
| 1 | **No early engagement, no out-of-network reach.** With 1,143 followers, growth has to come from out-of-network candidate sourcing, which depends on engagement signals that the posts never receive. | Reach observed; mechanism verified in the 2023 SimClusters design, where post embeddings are built from favorites |
| 2 | **The ranker rewards conversation, and xbot earns none.** Replies are weighted far above likes in both the 2023 and the 2026 weights, and the live defaults add heavy weights on quotes and shares. The audited posts have 0 replies. | Verified in published code (direction); exact multipliers have changed |
| 3 | **Account-level reputation may be dampened.** 9,125 posts with almost no engagement could depress the account's standing in graph-based scoring. | Hypothesis |
| 4 | **The cadence is repetitive.** Repeat posts by one author are score-discounted within a request, and repetitive micro-updates invite mutes. | Verified mechanism; small effect at current reach |
| 5 | **Cashtags are absent.** Posts without `$BTC` or `$ETH` do not appear in cashtag searches. | Plausible; effect on For You distribution is not documented |

Section 4 adds five operational findings: a posting dormancy since October 15, a Python 3.13+ import crash, X API 403 errors on quote-posts, a MongoDB connection with no timeout, and an unconfirmed "Automated" label.

**Recommended first actions**

1. Restore posting and diagnose the dormancy.
2. Confirm the account's "Automated" label.
3. Pin a macro-thesis thread to the profile.
4. Ship cashtags and grounded conversational hooks.
5. Alert the operator when a post goes live, so early replies get answered.
6. Consolidate routine micro-posts into 8-hour shift reports.

---

## 1. System Architecture and Content Pipeline

### 1.1 Purpose and identity

The bot automates content delivery for **TitanTech Investments** (`@TitanTechIn`), a Dubai-based entity whose stated mandate is *"Building the Future of Finance | Global reach, expertise in Crypto, Trading, Tech, AI."*

Unlike typical promotional crypto bots that rely on hype keywords, emojis and hashtags, xbot enforces a **restrained, cynical, institutional persona**:

- **Tone:** dry, deadpan, concise (under 270 characters for short takes).
- **Style:** standard English capitalization and punctuation; no hashtags, emojis, exclamation marks or hype phrases (`lambo`, `moon`, `lfg`, `bullish af`).
- **Grounding:** every post is anchored in verified data points (exact liquidation notionals, basis points, 24-hour level crossings, funding-rate deltas). Speculative superlatives ("first time ever", "historic record") are prohibited unless they appear in the source data.

![Figure 2. The xbot pipeline: ingestion, decision, generation and action, feeding six content streams](images/fig02_xbot_pipeline.png)

*Figure 2. The xbot pipeline: ingestion, decision, generation and action, feeding six content streams.*

### 1.2 Core content streams

| # | Content stream | Source / trigger | Generation and pipeline | Cadence / cap |
|---|---|---|---|---|
| 1 | **Market takes** | Binance futures WebSocket and REST | Quantitative triggers: funding flips, liquidation cascades, dynamic key-level breaks, open-interest deltas | 2 reserved daily slots; up to 5/day cap |
| 2 | **News reactions** | Top RSS news feeds | Gemini 2.5 Flash filters for significance and relevance, then writes an original take (no verbatim repeats) | Gated by a 30-minute cooldown; stops at cap minus 2 |
| 3 | **Data infographics** | Binance futures and CoinGecko | Algorithmic Pillow rendering across 7 sibling layouts (`fused_pie`, `ring`, `split_bars`, and others) | 1 standalone per day (14:00 UTC), plus attached to data posts |
| 4 | **Long-form articles** | Ingested weekly news and market concepts | 2,200 to 3,500 character analytical deep dives (Sundays: News Roundup; Wednesdays: Mechanism Explainer) | 2 per week; outside the daily post budget |
| 5 | **Curated engagement** | Whitelist of 25+ top analysts | Native reposts of high-signal posts; drafts quote-post counterpoints | Up to 3 reposts and 3 comment drafts per day |
| 6 | **Memes** | Sharp price moves and mood shifts | Deterministic router: text take (55%), Imgflip template (30%) or Gemini AI visual (15%) | Capped at 1 to 2 per day |

---

## 2. Quantitative Results and Live Production Audit

### 2.1 Profile metrics

Inspection of the live profile at [x.com/titantechin](https://x.com/titantechin) shows:

| Metric | Value |
|---|---|
| Total posts | 9,125 |
| Followers | 1,143 |
| Following | 225 |
| Follower-to-following ratio | about 5.1 : 1 |
| Most recent post | October 15, 2026 (5 days before the audit) |
| Unit tests | 184 passing |

The historical cadence is continuous automated output covering funding flips, level retests, liquidations, daily infographics and articles.

### 2.2 Recent live production samples

| # | Date | Format | Views | Likes | Replies |
|---|---|---|---|---|---|
| 1 | October 15, 2026 | Infographic take (1600 × 1600 card) | 33 | 0 | 0 |
| 2 | October 15, 2026 | Level break and funding flip (1600 × 1600 chart) | 33 | 0 | 0 |
| 3 | October 15, 2026 | Liquidation cascade (1600 × 1600 chart) | 27 | 0 | 0 |
| 4 | Earlier (undated) | Raw micro-update (dense text) | 28 | 0 | 0 |
| 5 | Earlier (undated) | Re-test update (dense text) | 31 | 0 | 0 |

**Sample 1** (infographic take):
> *"More than half of the sector's value remains concentrated in the first asset, despite years of capital flowing into alternatives."*

**Sample 2** (level break and funding flip):
> *"BTCUSDT dropped to 79876.61, a level it has crossed seven times in the last 24 hours. Funding has flipped negative to -0.0004% per 8h, while open interest rose 1.01% in 15 minutes, making new shorts expensive."*

**Sample 3** (liquidation cascade):
> *"ETHUSDT saw \$13.3M in long liquidations in five minutes at 2345.19. Funding climbed to 0.0054% per 8h from -0.0017%, making new long entries more expensive as price retests the 24-hour low."*

**Sample 4** (raw micro-update, excerpt):
> *"ETHUSDT funding flipped to 0.0082% per 8h at 2421, up from -0.0009%... of the 24-hour range."*

**Sample 5** (re-test update):
> *"BTCUSDT funding is now 0.0033% making longs expensive as price retests the 24-hour high."*

**What the samples show**

- Every post sits in a narrow band of 27 to 33 views. The mean of 30.4 views is about 2.7% of the follower count. Views are not unique followers, so this is indicative only.
- No post has a single reply, so no conversation signal exists for the ranker to use.
- Posts with an attached card (27 to 33 views) did not outperform raw-text posts (28 to 31 views). Five posts is a small sample, but it suggests that a visual layer alone will not fix reach. The bottleneck appears to sit upstream, in distribution.

---

## 3. Reverse-Engineering X's Algorithm

### 3.1 Two generations of the algorithm

The open-sourced recommendation code shows how the For You timeline is built. Two public releases matter here, and the live one is not the one the audit started from.

![Figure 3. The 2023 reference pipeline compared with the live 2026 pipeline](images/fig03_algorithm_pipelines.png)

*Figure 3. The 2023 reference pipeline compared with the live 2026 pipeline.*

- **Reference used in this audit:** [`twitter/the-algorithm`](https://github.com/twitter/the-algorithm), open-sourced in March 2023. It sources candidates, scores them with the HeavyRanker (a parallel MaskNet neural network), then applies filtering heuristics and mixing.
- **Live system:** [`xai-org/x-algorithm`](https://github.com/xai-org/x-algorithm), open-sourced in January 2026 and still updated (its README lists changes through September 2026). It ranks posts with a Grok-based transformer called Phoenix and combines in-network and out-of-network posts in a single ranking.

The mechanics below are mapped against both. Where they differ, the live system governs what happens to a post today.

### 3.2 Candidate sourcing

**In-network.** The 2023 system sources posts from followed accounts through Earlybird; the live system uses Thunder, which keeps recent posts from followed accounts in memory. With 1,143 followers, in-network reach is bounded by the size of the follower base and by how many followers are active. The observed 27 to 33 views per post is consistent with that ceiling.

**Out-of-network.** The 2023 system relied on embeddings from SimClusters and TwHIN. The live system uses Phoenix retrieval, SimClusters and a shared list of popular recent posts. SimClusters clusters accounts into overlapping communities from the follow graph (the 2023 documentation describes about 145,000 of them) and represents posts as vectors over those communities. In the 2023 design, roughly half of the For You feed came from in-network sources and the rest from out-of-network ones.

**Why xbot stalls.** In the 2023 SimClusters design, a post's embedding starts as an empty vector and is updated each time the post is favorited, by adding the interests of each user who liked it. A post with zero likes therefore has nothing for community-based retrieval to match on, and all five audited posts have zero likes. The live README describes SimClusters as clustering accounts and posts by who engages with what, which points the same way, but this edition has verified the favorites-driven mechanism only in the 2023 code. Two live-system rules compound the problem: out-of-network posts are multiplied by a factor below 1, and an `AgeFilter` removes posts older than 48 hours, so a post that does not catch on early rarely gets a second chance.

Two notes on precision:

- An earlier edition of this report cited a 30-minute cut-off for entering SimClusters pools. No such threshold was confirmed in the published code; the verified mechanism is that embeddings grow with each favorite, so this edition relies on early engagement in general rather than a specific window.
- The live system also contains a **New-Author Boost**, which lifts posts from authors whose impressions fall below a threshold. The README does not state the threshold, so whether @TitanTechIn qualifies is unknown.

### 3.3 The ranker scoring deficit

Both generations compute a composite score as a weighted sum of predicted engagement probabilities:

```
Final score = Σ weight_i × P(action_i)
```

**April 2023 snapshot (the audit's reference).**

![Figure 4. HeavyRanker weights from the April 2023 snapshot](images/fig04_weights_2023.png)

*Figure 4. HeavyRanker weights from the April 2023 snapshot. Multiples are relative to a like (0.5).*

| Signal | Weight | Multiple of a like | What the model predicts (2023 README) | xbot today |
|---|---|---|---|---|
| **Reply engaged by author** | +75.0 | 150× | The user replies to the post and the author engages with that reply | None observed (0 replies; no reply-back mechanism) |
| **Reply** | +13.5 | 27× | The user replies to the post | 0 replies on audited posts |
| **Good profile click** | +12.0 | 24× | The user opens the author's profile and likes or replies to a post | Not measured |
| **Good click** | +11.0 | 22× | The user clicks into the conversation and replies or likes | Not measured |
| **Good click v2** | +10.0 | 20× | The user clicks into the conversation and stays at least 2 minutes | Not measured |
| **Retweet** | +1.0 | 2× | The user retweets the post | Not reported in the samples |
| **Like** | +0.5 | 1× (baseline) | The user likes the post | 0 likes on audited posts |
| **Negative feedback** | −74.0 | −148× | "Show less often", block or mute | Not measured; repetitive posting raises the risk |
| **Report** | −369.0 | −738× | The user reports the post | Not measured |

**October 2026 live defaults.**

![Figure 5. Live weight defaults from xai-org/x-algorithm](images/fig05_weights_2026.png)

*Figure 5. Live weight defaults declared in `home-mixer/params/param.rs` (read October 20, 2026).*

| Signal | April 2023 | October 2026 default | Note |
|---|---|---|---|
| Like | 0.5 | 0.66 | |
| Repost / retweet | 1.0 | 1.0 | |
| Reply | 13.5 | 5.0 | |
| Reply engaged by author | 75.0 | No direct counterpart | A separate `BidirectionalFollowReplyWeightBoost` (default 15.0) exists; read its definition in `param.rs` before relying on it |
| Profile click | 12.0 (profile visit plus like or reply) | 0.0 | Follow author is weighted +4.0 |
| Conversation click and dwell | 11.0 and 10.0 | Click 0.3; dwell-related terms 0.05, 0.006 and 0.4 | Different definitions, partly predicted dwell time rather than a click probability; not comparable one to one |
| Quote | Not listed | 5.0 | |
| Share, share via DM, share via copy link | Not listed | 2.0, 5.0 and 20.0 | |
| Negative feedback | −74.0 | Not interested −52.272; mute author −58.8; block author −31.2 | |
| Report | −369.0 | −234.0 | |

**What holds in both generations**

- A like is the cheapest positive signal.
- A reply is worth many times a like (13.5 against 0.5 in 2023; 5.0 against 0.66 today), and the live defaults also weight quotes, shares and follows several times a like.
- Negative feedback and reports carry large penalties, so repetitive posts that get muted are costly.
- Weights apply to *predicted probabilities*, which depend on each viewer's own behavior. X's README explicitly warns against reading weight ratios as raw engagement-count equivalents.

**Key takeaway.** In the 2023 weights the single highest-weighted positive signal was the author engaging with a user's reply (+75.0). xbot is an unmonitored broadcast script with no reply-back mechanism, so it earned none of it. The live defaults have no direct equivalent, so the +75.0 figure should not be a target. The practical lesson is broader and still valid: earn replies, quotes and shares, and answer people who reply. Treat prompt author replies as a low-cost experiment, not a guaranteed multiplier.

### 3.4 Account reputation: TweepCred and its successor

In the 2023 code, TweepCred runs a PageRank-style algorithm over the interaction graph (mentions, retweets and similar edges) and scales the result to a 0 to 100 score. The live repository's counterpart, `user-cred-v2`, runs PageRank over the follow graph and engagement edges and converts the resulting mass into a per-account score. The live repository also contains bot- and spam-labelling systems (`botmaker`, `scarecrow`, `bdsm`) whose labels feed visibility filtering.

What the 2023 code does and does not penalize:

- The account's base "mass" depends on factors such as device, account age and verified, restricted or suspended status.
- A dampening step applies only to accounts that follow many more people than follow them back: more than 2,500 followings in `Reputation.scala` (more than 500 and a following-to-follower ratio above 0.6 in `UserMass.scala`).
- **@TitanTechIn does not trigger that penalty.** It follows 225 accounts against 1,143 followers, a ratio of about 0.20.
- The code contains no term for engagement per post, so the claim that 9,125 posts with fewer than 0.1 engagements each would "severely dampen" TweepCred is not supported by the 2023 source.

**Assessment (hypothesis).** Because the live `user-cred-v2` uses engagement edges, a long record of posts with near-zero engagement could plausibly leave the account with a weak score, and unlabelled automation could attract bot-labelling attention. Neither can be confirmed from outside the platform. This is a reason to improve engagement and confirm the "Automated" label, not a proven cause of the stall.

### 3.5 Author diversity

The 2023 `home-mixer` applies an `AuthorDiversityDiscountProvider`. Within the candidates scored for a request, each author's posts are sorted by score and the n-th best post is multiplied by an exponentially decaying discount with a floor (decay 0.5, floor 0.25):

```
discount(position) = (1 − 0.25) × 0.5^position + 0.25
```

| Position of the post among one author's candidates | Score multiplier |
|---|---|
| 1st | 1.000 |
| 2nd | 0.625 |
| 3rd | 0.4375 |
| 4th | 0.34375 |
| Floor | 0.25 |

The live system keeps the same idea: each post after an author's first is multiplied by a decaying factor, down to a floor.

An earlier edition called this an `AuthorDiversityFilter` that "coarse-grained drops" posts. In the code it is a score discount, not a drop. The effect on xbot is small at roughly 30 views per post, but it grows with reach, and when several micro-updates (5 to 8 per day, as estimated in the audit) land in the same viewer's candidate set, the later ones are penalized. Repetitive updates also raise the chance of "not interested" or mute actions, which carry the heaviest penalties.

### 3.6 Where xbot's posts drop out

| Stage | What happens | xbot symptom | Evidence status |
|---|---|---|---|
| Candidate sourcing | Out-of-network retrieval needs an engagement signal to match a community | 27 to 33 views, 0 engagement | Reach observed; mechanism plausible |
| Ranking | The weighted sum rewards replies, quotes, shares and follows | 0 replies | Verified (direction) |
| Attention signals | Click and dwell terms add to the score | Short raw text is read in seconds; no dwell data in the audit | Plausible |
| Account reputation | A graph-based per-account score | 9,125 posts with almost no engagement | Hypothesis |
| Diversity discount | Repeat posts by one author are discounted | Several micro-updates per day | Verified mechanism; small effect at current reach |

---

## 4. Technical System Issues

Beyond ranking, the audit of the local repository (`file:///Users/ninja/Downloads/xbot`) found five operational issues.

| ID | Issue | Impact | Recommended fix |
|---|---|---|---|
| T1 | **Python 3.13+ import crash.** `requirements.txt` pins `tweepy==4.14.0`, whose `api.py` imports the standard-library `imghdr` module at import time. Python 3.13 removed `imghdr`. | On Python 3.13 or 3.14 the test suite and the production entrypoint fail immediately with `ModuleNotFoundError: No module named 'imghdr'`. | Upgrade tweepy to a release that guards the import (4.17.0 does, verified), or pin the interpreter to Python 3.12 or earlier. |
| T2 | **403 Forbidden on quote-posts.** `src/xbot/config.py:249` defaults `comment_approval` to `False`, so `src/xbot/feeds/commenter.py:199` tries direct API quote-posts of accounts that have not mentioned the bot. | X rejects the call on the Basic tier with: `403 Forbidden: You can only reply to or quote posts where you are mentioned or are the author.` This is consistent with X's February 2026 restriction on programmatic replies for self-serve API tiers. | Default `comment_approval` to `True` (Section 6.4) so drafts go to the operator for manual posting; handle 403 responses without retry loops. |
| T3 | **5-day production dormancy.** The latest post on @TitanTechIn is dated October 15, 2026. The EC2 service (`ubuntu@13.204.84.120`) has halted, run out of Gemini or X API credits (status 402), or crashed. | No content is published, so no audience is built. | Check service logs, credit balances and the host's Python version (see T1) first. The cause is undiagnosed. |
| T4 | **MongoDB Atlas connection lockout.** [`Database.connect()`](file:///Users/ninja/Downloads/xbot/src/xbot/db/client.py#L38) sets no timeouts. | Running check scripts from outside the Atlas IP allowlist hangs for about 30 seconds (the driver's default server-selection timeout) before failing. | Pass explicit `serverSelectionTimeoutMS` and `connectTimeoutMS` values when constructing the client. |
| T5 | **Unconfirmed "Automated" label.** `deploy/DEPLOY.md` line 104 shows the X "Automated" label unchecked. | X's automation rules expect automated accounts to be labelled; operating without the label risks enforcement action, up to suspension. | Enable the label in account settings and tick the checklist item. |

---

## 5. The Algorithmic Engineering Blueprint

Five pillars realign xbot with how X's recommendation system rewards content.

![Figure 6. The five-pillar blueprint](images/fig06_blueprint_pillars.png)

*Figure 6. The five-pillar blueprint: what changes and which behavior each change is meant to move.*

### Pillar 1: Mandatory visual layer

- **Rationale.** High-resolution cards stop the scroll and invite taps, which feeds the click and dwell terms. These terms were large in the 2023 weights (+10.0 to +11.0) and are small in the live defaults (click 0.3). The case is therefore weaker today than the original audit assumed.
- **Evidence caveat.** In the audited sample, posts with cards did not outperform raw text (Section 2.2). Keep the visual layer for brand consistency and scannability, but do not expect it to lift reach on its own.
- **Rule.** No raw-text posts. If the 7-layout infographic engine cannot extract a structured dataset, fall back to an automated 24-hour candlestick chart (Matplotlib).

### Pillar 2: 8-hour market shift reports

- **Rationale.** Micro-ticks that earn no engagement add little, trigger the author-diversity discount and raise mute risk.
- **Rule.** Consolidate routine updates into **three comprehensive daily reports** timed to Binance funding settlements at **00:00, 08:00 and 16:00 UTC**. Each report synthesizes price action, funding-rate drift, open-interest delta and liquidation totals into one visual.
- **Exception.** Reserve instant posting for **critical outliers**: liquidations above \$25M or major level breaks.

![Figure 7. Shift-report slots on a 24-hour UTC clock against the operator presence window](images/fig07_daily_schedule.png)

*Figure 7. Shift-report slots on a 24-hour UTC clock against the default operator presence window.*

> **Scheduling conflict to resolve.** The default presence window in Section 6.3 is 07:00 to 21:00 UTC, but one of the three reports is scheduled for 00:00 UTC, outside it. Either exempt scheduled reports from the window check or widen the window to cover 00:00.

### Pillar 3: Conversational hooks

- **Rationale.** Replies carry far more weight than likes in both generations (13.5 vs 0.5 in 2023; 5.0 vs 0.66 today), and the 2023 weights gave a further +75.0 when the author engaged with a reply.
- **Rule.** Modify `SYSTEM_PERSONA` in [`src/xbot/persona.py`](file:///Users/ninja/Downloads/xbot/src/xbot/persona.py) so that every take ends with an open trade tension that invites debate, using only data already present in the post.

| Passive format (invites no reply) | Conversational hook (invites replies) |
|---|---|
| *BTCUSDT funding is now 0.0033% making longs expensive as price retests the 24-hour high.* | *\$BTC funding is now 0.0033% per 8h, making longs expensive as price retests the 24-hour high. Is this a squeeze setup or a late-long flush?* |
| *ETHUSDT saw \$13.3M in long liquidations in five minutes at 2345.19.* | *\$ETH saw \$13.3M in long liquidations in five minutes at 2345.19, with funding at 0.0054% per 8h. Is this capitulation, or are late longs still exposed as price retests the 24-hour low?* |

The hook examples were reworded for this edition. The earlier versions added claims that are not in the source data (a "\$80k resistance" level, funding that had "fully reset", longs "wiped out"), which would breach the persona's own grounding rule.

**Operational note.** X's February 2026 restriction means that on self-serve API tiers, API-posted replies are accepted only in conversations where the other account has mentioned or quoted the bot. Replies from the operator in the X app are unaffected. Any automated reply-to-replier feature needs to be tested against this restriction before it is relied on; the alert in Section 6.5 supports the manual route.

### Pillar 4: Profile anchors and deep threads

- **Rationale.** In the 2023 weights, a profile visit followed by a like or reply scored +12.0. In the live defaults the plain profile click is 0.0, but a follow is worth +4.0. Visitors follow when they find structured research on arrival.
- **Rule.** Replace the generic bio slogan with a pinned, high-value macro-thesis thread. Convert the Wednesday Mechanism Explainer into a **5-post visual thread** (`1/5` to `5/5`).

### Pillar 5: Cashtag search discovery

- **Rationale.** Cashtags (`$BTC`, `$ETH`, `$SOL`) are X's native ticker entities. A post that contains only the bare text "BTC" does not appear when traders search the cashtag. Whether cashtags change For You distribution is not documented in the published ranking code, so this is a search-discovery lever, and its reach effect should be measured.
- **Rule.** Cashtag the relevant tickers automatically (Section 6.1). As a guideline, keep it to one or two cashtags per post to avoid a spam appearance.
- **Persona check.** Cashtags are not hashtags, but the persona forbids hashtags and the voice guard may enforce that with a pattern that also catches `$`-prefixed tokens. Confirm that cashtags pass it.

---

## 6. Code Modifications

### 6.1 Cashtag auto-injection: [`src/xbot/persona.py`](file:///Users/ninja/Downloads/xbot/src/xbot/persona.py)

The first-edition version of this function turned an existing `$BTC` into `$$BTC`, because `\b` matches between `$` and `B`. The corrected version below leaves existing cashtags and longer words (`Ethereum`, `solid`) alone, and is idempotent.

```python
import re

_CASHTAGS = {
    "btcusdt": "$BTC", "btc": "$BTC",
    "ethusdt": "$ETH", "eth": "$ETH",
    "solusdt": "$SOL", "sol": "$SOL",
}
# Longest keys first so "btcusdt" is tried before "btc". The lookbehind skips text
# that is already cashtagged ("$BTC") or embedded in a longer token; the lookahead
# requires the ticker to end at a word boundary.
_CASHTAG_RE = re.compile(
    r"(?<![\w$])("
    + "|".join(sorted(map(re.escape, _CASHTAGS), key=len, reverse=True))
    + r")(?!\w)",
    re.IGNORECASE,
)


def inject_cashtags(text: str) -> str:
    """Cashtag bare tickers for X search. Idempotent; the USDT pair suffix is dropped."""
    return _CASHTAG_RE.sub(lambda m: _CASHTAGS[m.group(1).lower()], text)
```

Suggested unit tests (all verified against the code above):

```python
def test_inject_cashtags():
    assert inject_cashtags("BTCUSDT dropped to 79876.61") == "$BTC dropped to 79876.61"
    assert inject_cashtags("$BTC retests $80k") == "$BTC retests $80k"
    assert inject_cashtags("Ethereum and solid funding") == "Ethereum and solid funding"
    assert inject_cashtags("btc, eth and SOL") == "$BTC, $ETH and $SOL"
    text = "BTCUSDT and $ETH and sol"
    assert inject_cashtags(inject_cashtags(text)) == inject_cashtags(text)
```

### 6.2 Conversational hook prompt: `SYSTEM_PERSONA` ([`src/xbot/persona.py:85-100`](file:///Users/ninja/Downloads/xbot/src/xbot/persona.py#L85-L100))

```python
SYSTEM_PERSONA = """...
substance rules:
- State what happened, what it means for positioning, and end with a sharp
  desk question that invites active traders to debate (for example, local top
  versus breakout, squeeze versus capitulation).
- Frame a genuine trade tension: who is trapped, what level forces liquidation,
  or whether the basis is mispriced. This drives replies and early engagement.
- The question must rest only on data already in the post. Never add levels,
  causes or superlatives that are not in the source data.
"""
```

The final rule is new in this edition. It keeps the hooks consistent with the persona's grounding requirement.

### 6.3 Active presence window: [`src/xbot/decision/arbiter.py`](file:///Users/ninja/Downloads/xbot/src/xbot/decision/arbiter.py)

Non-critical posts drop only while the operator is awake, so early replies can be answered. The corrected version handles windows that wrap past midnight and converts any timezone-aware `datetime` to UTC.

```python
from datetime import datetime, timezone

ACTIVE_HOURS_UTC: list[int] = Field(default=[7, 21], alias="ACTIVE_HOURS_UTC")  # [start, end), UTC hours


def _in_presence_window(self, now: datetime) -> bool:
    """True when `now` falls inside the operator's waking hours (UTC, end-exclusive)."""
    start, end = self._s.active_hours_utc
    hour = now.astimezone(timezone.utc).hour
    if start <= end:
        return start <= hour < end
    return hour >= start or hour < end  # window wraps past midnight
```

Pass timezone-aware datetimes; a naive `datetime` is interpreted as local time. Set the defaults to the operator's real waking hours, and see the scheduling conflict in Pillar 2 for the 00:00 UTC report.

### 6.4 Config safety: [`src/xbot/config.py:249`](file:///Users/ninja/Downloads/xbot/src/xbot/config.py#L249)

```python
comment_approval: bool = Field(True, alias="COMMENT_APPROVAL")
```

With approval on, comment drafts go to the operator for manual posting rather than being sent through an API call that X rejects with a 403 (issue T2).

### 6.5 Post-drop operator alert: [`src/xbot/action/engine.py:325`](file:///Users/ninja/Downloads/xbot/src/xbot/action/engine.py#L325)

Send a Telegram or Discord alert the moment a post is published, so the operator can answer early replies. The alert must never block or break the posting path.

```python
if tweet_id:
    try:
        await notify_operator(
            f"Live on @TitanTechIn: https://x.com/TitanTechIn/status/{tweet_id}\n"
            "Early replies are worth answering promptly."
        )
    except Exception:
        logger.warning("operator alert failed", exc_info=True)  # use the module's logger
```

This assumes a `notify_operator` helper that posts to the chosen webhook.

---

## 7. Roadmap and Priority Matrix

![Figure 8. Roadmap and priority matrix](images/fig08_roadmap_matrix.png)

*Figure 8. Eight milestones across four phases.*

| Phase | Milestone | Objective | Target |
|---|---|---|---|
| **Immediate (Day 1)** | Production health check | Resolve the posting dormancy since October 15 (`ubuntu@13.204.84.120`) | Restore heartbeat |
| **Immediate (Day 1)** | Policy compliance | Set the "Automated" label in account settings | Avoid enforcement action |
| **Immediate (Day 1)** | Pin a macro anchor | Pin an analytical macro thread to the profile | Profile visits and follows |
| **Immediate (Day 1)** | Technical fixes | Fix T1, T2 and T4: tweepy upgrade or Python pin, comment approval on, MongoDB timeouts | Stable runtime |
| **Short-term (Days 2 to 3)** | Cashtags and trade tension | Deploy the cashtag function and conversational hook prompts | Search reach and replies |
| **Short-term (Days 2 to 3)** | Post-drop alerts | Send a Telegram or Discord alert on publish | Prompt replies to early commenters |
| **Medium-term (Week 1)** | 8-hour settlement reports | Consolidate micro-posts into comprehensive shift updates | Fewer low-engagement posts |
| **Medium-term (Week 2)** | Visual thread engine | Convert weekly articles into 5-part visual threads | Dwell time and profile visits |

### Measuring success

The core problem is distribution, so measure distribution. Record baselines before any change, then compare after each phase.

| Metric | Baseline (October 2026) |
|---|---|
| Views per post | Mean 30.4 (range 27 to 33) |
| Replies per post | 0 |
| Likes per post | 0 |
| Followers | 1,143 |
| Profile visits, net follows per day, share of impressions from non-followers | Not captured; record from X analytics before changing anything |

Introduce the changes in time-boxed blocks, rather than all at once, so that any improvement can be attributed to a specific change.

---

## Appendix A: Evidence and Confidence

This edition re-checked the audit's technical claims against the published sources. The table lists what changed from the first edition and why.

| Claim in the first edition | Status in this edition | Basis |
|---|---|---|
| Weights of +75.0, +13.5, +12.0, +11.0, +1.0, +0.5, −74.0 and −369.0 | Verified for April 2023 only | `the-algorithm-ml` recap README; superseded by the live `param.rs` defaults |
| "+11.0 is a conversation click with 2-minute dwell" | Corrected | +11.0 is a conversation click followed by a reply or like; the 2-minute dwell term is +10.0 |
| "+12.0 is a profile click or follow" | Corrected | The 2023 term is a profile visit followed by a like or reply |
| TweepCred penalizes low engagement per post | Not supported by the 2023 code | The dampening applies to high following-to-follower ratios; @TitanTechIn's ratio is about 0.20 |
| `AuthorDiversityFilter` drops posts | Corrected | `AuthorDiversityDiscountProvider` applies a score discount (decay 0.5, floor 0.25) |
| 30-minute SimClusters cut-off | Replaced | No threshold found; the verified 2023 mechanism is that post embeddings are updated on each favorite |
| Visual layer captures the dwell boost | Weakened | Dwell and click terms are small in the live defaults; cards did not outperform raw text in the sample |
| Cashtags are needed for "indexing" | Softened | Cashtag searches match cashtag entities; effect on For You distribution is undocumented |
| tweepy 4.14.0 crashes on Python 3.13+ | Verified | `api.py` imports `imghdr` at module level; 4.17.0 guards the import |
| Cashtag regex | Bug fixed | The first-edition regex produced `$$BTC` from an existing `$BTC` |
| Hook examples | Reworded | Ungrounded claims removed to match the persona's grounding rule |
| 403 on quote-posts | Contextualized | Consistent with X's February 2026 restriction on programmatic replies for self-serve tiers |
| 00:00 UTC report vs the 07:00 to 21:00 UTC presence window | New finding | The two settings contradict each other |

---

## Appendix B: Sources

- **HeavyRanker weights (April 2023):** [twitter/the-algorithm-ml, `projects/home/recap/README.md`](https://github.com/twitter/the-algorithm-ml/blob/main/projects/home/recap/README.md)
- **Author-diversity discount (2023):** [`DiversityDiscountProvider.scala`](https://github.com/twitter/the-algorithm/blob/main/home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/scorer/DiversityDiscountProvider.scala)
- **TweepCred (2023):** [`Reputation.scala`](https://github.com/twitter/the-algorithm/blob/main/src/scala/com/twitter/graph/batch/job/tweepcred/Reputation.scala) and [`UserMass.scala`](https://github.com/twitter/the-algorithm/blob/main/src/scala/com/twitter/graph/batch/job/tweepcred/UserMass.scala)
- **Live system (2026):** [xai-org/x-algorithm README](https://github.com/xai-org/x-algorithm) and [`home-mixer/params/param.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/params/param.rs), both read October 20, 2026
- **January 2026 open-sourcing:** [Decrypt coverage](https://decrypt.co/355108/elon-musks-x-open-sources-grok-powered-algorithm-driving-for-you-feed)
- **X API reply restriction (February 2026):** [Roboin coverage](https://roboin.io/article/en/2026/02/24/x-limits-api-based-automated-replies-to-combat-engagement-farming/)
- **tweepy source:** release 4.14.0 and release 4.17.0 from PyPI, `tweepy/api.py`

The weight values in the live system are defaults declared in the repository and can be overridden by runtime configuration.

---

## Appendix C: Glossary

| Term | Meaning |
|---|---|
| **Cashtag** | A `$`-prefixed ticker such as `$BTC` that X treats as a searchable entity. |
| **Earlybird** | The 2023 in-network candidate source for posts from followed accounts. |
| **HeavyRanker** | The 2023 neural ranker (a parallel MaskNet) that predicts engagement probabilities. |
| **Phoenix** | The live Grok-based transformer that retrieves and ranks candidates. |
| **SimClusters** | Groups accounts into overlapping communities and represents posts as vectors over them; used to find out-of-network candidates. |
| **Thunder** | The live in-network source; holds recent posts from followed accounts in memory. |
| **TweepCred** | The 2023 PageRank-based account reputation score (0 to 100); `user-cred-v2` is the live counterpart. |
| **TwHIN** | Dense knowledge-graph embeddings for users and posts, used alongside SimClusters in the 2023 system. |
