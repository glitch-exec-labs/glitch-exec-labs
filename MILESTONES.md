# Glitch Executor — Milestones

Lab-level record of what we've shipped, why we built it, and what we learned.
This file is mined by our social media agent to generate build-in-public content.

---

## Priya — Bilingual COD Confirmation Voice Agent
**Domain:** Edge / Voice AI  
**Repo:** glitch-cod-confirm  
**Status:** Live in production  
**Date:** 2026-Q1

India's D2C e-commerce problem: COD (cash on delivery) orders have 30–40% RTO (return-to-origin) rates. A customer places an order, never answers the door, and the courier returns it — you eat the shipping cost twice. The standard fix is a human call center. We replaced it.

Priya calls customers within minutes of order placement, confirms intent, and flags likely-to-RTO orders before dispatch. The hard part was language. Indian customers hang up on English-only IVR. We needed native Hindi phoneme quality — not text-to-speech with an accent, actual Hindi. That ruled out Deepgram and ElevenLabs. We chose Sarvam AI's Bulbul v3 model, built specifically for Indian languages. LiveKit handles the real-time WebRTC layer — latency matters when someone's answering the phone. GPT-4o-mini runs the conversation logic.

The system is multi-tenant by design. Per-shop prompts, voices, and languages are driven by Shopify metafields, so one deployment serves many stores without code changes.

**Key decisions:** Sarvam over Western TTS providers for Hindi quality. LiveKit over Twilio for latency. GPT-4o-mini over GPT-4 for cost at call volume.  
**What we learned:** The failure mode isn't the AI — it's the phone network. Indian PSTN drops calls unpredictably. We built retry logic and a fallback SMS path before we knew we needed it.  
**Signal tags:** voice-ai, edge, production, shopify, india, rto, cod, hindi, bilingual, livekit, sarvam

---

## Ouroboros — Flagship Multi-Bot Trading Ensemble
**Domain:** Trade  
**Repo:** glitch-ouroboros-snake-strategy  
**Status:** Active  
**Date:** 2026-Q1

The core thesis: no single trading model works across all market conditions. A momentum strategy prints in trending markets and gets chopped in ranging ones. Mean reversion does the opposite. The question is how to coordinate them.

Ouroboros is a multi-bot ensemble where an Oracle layer monitors all active strategies, tracks correlation between their positions, and enforces portfolio-aware risk controls. If Viper (momentum) and Cobra (breakout) are both long the same instrument, the Oracle reduces total exposure — they're not independent bets. Each individual bot doesn't know the others exist. The Oracle sits above and manages the ensemble as a portfolio.

The snake naming isn't decoration. Each snake has a distinct attack pattern that maps to a trading personality: Viper is fast and momentum-driven, Anaconda is slow and constricting (mean reversion), Hydra regenerates from losses with adaptive sizing. The metaphor keeps the strategy logic memorable across the team.

Broker-portable by design — the abstraction layer supports MT5, cTrader, and Interactive Brokers without strategy code changes.

**Key decisions:** Oracle coordination over shared state. Broker abstraction from day one. Named personalities per bot to keep strategy intent clear.  
**What we learned:** Correlation between strategies matters more than individual Sharpe ratios. Two great strategies that correlate at 0.9 add less value than one good strategy that's uncorrelated.  
**Signal tags:** trade, algotrading, ensemble, oracle, mt5, ctrader, quantfinance, riskmanagement, multibot

---

## Indian King Cobra — Standalone Momentum Bot
**Domain:** Trade  
**Repo:** glitch-indian-king-cobra  
**Status:** Active  
**Date:** 2026-Q1

Momentum strategies have a problem: they enter on strength, but news events create false breakouts. A stock gaps up 3% on earnings — momentum says buy, but it's already priced in and about to reverse. We built news-aware filtering to gate entries.

Indian King Cobra runs standard momentum signals (price, volume, relative strength) but adds two gates before entry: an ML classifier that scores the signal quality, and a news filter that checks whether the momentum is happening during a known event window (earnings, macro releases, FOMC). If both gates clear, the entry fires. If either flags, it skips.

The ML gate was trained on our own historical signals — which ones led to winning trades versus which led to getting stopped out within the first hour. It learned to recognize the "good momentum" signature from the bad one.

**Key decisions:** Two gates (ML + news) rather than one to reduce false positives. Train on our own signal history, not generic market data.  
**What we learned:** The news filter alone catches 60% of the bad entries. The ML gate is a refinement on top. Order matters — cheap filter first, expensive model second.  
**Signal tags:** trade, momentum, ml, newsfilter, algotrading, quantfinance, entries

---

## Terciopelo — Equities Relative Value Strategy
**Domain:** Trade  
**Repo:** glitch-terciopelo  
**Status:** Active  
**Date:** 2026-Q1

Named after the fer-de-lance viper — patient, precise, and dangerous when it strikes. Terciopelo is our equities strategy: relative value combined with mean reversion, filtered by technical scoring and news awareness.

Relative value means we're not betting on a stock going up — we're betting on it moving back toward its historical relationship with a peer or index. If Stock A and Stock B normally move together and they've diverged by 2.5 standard deviations, we bet on convergence. Mean reversion at the individual stock level handles cases where a stock has moved too far from its own historical range.

The news-aware layer was added after we got burned: a stock was 2 standard deviations cheap, our model said buy, and it was cheap because insider trading was being investigated. The news filter now checks for corporate events before we treat cheapness as an opportunity.

**Key decisions:** Added news gating after a live loss. Separate technical scoring layer to avoid fighting strong trends with mean reversion.  
**What we learned:** "Cheap" has two meanings — statistically cheap and fundamentally impaired. The model can't distinguish them without event data.  
**Signal tags:** trade, equities, relativevalue, meanreversion, algotrading, quantfinance, stocks

---

## Trading Core — Shared Trading Architecture
**Domain:** Trade  
**Repo:** glitch-trading-core  
**Status:** Active  
**Date:** 2026-Q1

When you have 8 trading bots, copy-pasting risk logic across each one is how you end up with 8 different bugs. Trading Core is the shared module layer that all our bots depend on: position sizing, risk limits, broker abstraction, Oracle coordination protocol, and event bus.

The decision to build this came after Viper and King Cobra each had their own position sizing implementations that differed by a rounding error. In live trading, rounding errors become real losses. We consolidated everything into a single tested implementation.

The MT5-to-cTrader migration is happening through this layer. Each bot calls `broker.open_position()` — the implementation underneath is swappable. We migrated two bots to cTrader without touching their strategy code.

**Key decisions:** Extract shared code when the second bot needs it, not the third. Broker abstraction as a first-class interface, not an afterthought.  
**What we learned:** The Oracle coordination protocol took three attempts to get right. First version used shared database state (too slow), second used message queues (too complex), third uses an in-process event bus with a 100ms heartbeat (simple enough to debug in live trading).  
**Signal tags:** trade, architecture, shared-modules, broker-abstraction, mt5, ctrader, refactoring, riskmanagement

---

## Cricket Engine — Live IPL/PSL Intelligence
**Domain:** Edge / Sports  
**Repo:** glitch-cricket-engine  
**Status:** Active  
**Date:** 2026-Q1

Cricket is the second-largest sports betting market in the world, mostly underserved by Western quant shops. The data is rich — ball by ball — but the modelling is harder than it looks. A T20 match can swing entirely on one over. Standard win probability models that don't account for match state momentum are too slow to be useful in-play.

The engine ingests ball-by-ball data and runs scenario models in real time: given this batter, this bowler, this field setting, and the current required run rate, what's the probability distribution of outcomes in the next 6 balls? That projection feeds into pricing decisions.

We paper-simulated for a full IPL season before going live. The paper simulation framework is built into the engine — it replays historical matches through the live code path so we could find pricing errors before they cost real money.

**Key decisions:** Ball-by-ball granularity over over-by-over. Paper simulation as a first-class testing mode, not an afterthought.  
**What we learned:** IPL and PSL have different signal characteristics. IPL pitches produce more pace-bowling dominance, PSL has more spin-friendly conditions. The engine needed separate calibration per competition, not one global model.  
**Signal tags:** edge, sports, cricket, ipl, psl, ml, inplay, pricing, scenario-modelling

---

## NBA Engine — Pregame Market Intelligence
**Domain:** Edge / Sports  
**Repo:** glitch-nba-engine  
**Status:** Active  
**Date:** 2026-Q1

NBA pregame pricing is a solved problem for major books — they have decades of data and sharp bettors closing every edge. We're not trying to beat that. The engine's job is market mapping: given the opening line and the current market consensus, where is the disagreement, and is that disagreement explainable by lineup context or unexplainable (i.e., potentially a mispricing)?

Lineup context is the key input. A team missing its starting centre plays differently than the same team at full strength — the spread should move, and if it hasn't fully moved, there's signal in that lag. The engine scrapes official injury reports and projects lineup impact on pace, defensive rating, and expected points.

**Key decisions:** Focus on the pregame window (2–4 hours before tip-off) when lineup info is confirmed but markets haven't fully adjusted. Don't compete with sharp books on the closing line.  
**What we learned:** The most reliable signal isn't injury news — it's the speed of market reaction to injury news. Fast market reaction = the line was already reflecting the risk. Slow reaction = potential lag to exploit.  
**Signal tags:** edge, sports, nba, basketball, pregame, lineups, marketpricing, injury-reports

---

## Betting Core — Shared Sports Intelligence Primitives
**Domain:** Edge / Sports  
**Repo:** glitch-betting-core  
**Status:** Active  
**Date:** 2026-Q1

Same problem as Trading Core, sports edition. The Cricket Engine and NBA Engine both need odds conversion, Kelly criterion position sizing, stake limits, and typed decision outputs. Betting Core is the shared library so each engine doesn't implement its own.

The typed decision output was the key design decision. Every engine returns a `BettingDecision` object with fields for edge estimate, confidence, stake recommendation, and reasoning. Downstream consumers (Telegram alert, paper simulator, live execution) all consume the same interface. Swapping the engine underneath doesn't break the consumer.

**Key decisions:** Kelly criterion for stake sizing with a fractional Kelly multiplier (0.3x) to reduce variance at the cost of some expected value. Typed outputs from day one.  
**What we learned:** The fractional Kelly multiplier needs to be lower than you think. Full Kelly is theoretically optimal but psychologically brutal during drawdowns — and drawdowns affect decision-making quality.  
**Signal tags:** edge, sports, kelly-criterion, stake-sizing, architecture, shared-modules

---

## Ads Agent — AI-Driven Ad Operations
**Domain:** Grow  
**Repo:** glitch-grow-ads-agent  
**Status:** Active  
**Date:** 2026-Q1

Managing Meta + Amazon + Shopify ads across multiple stores is a full-time job. The data is spread across three dashboards with no native cross-platform attribution. ROAS numbers lie because Meta pixel misses purchases that happen on Amazon, and Amazon attribution misses the Meta touchpoint that drove the awareness.

The ads agent does three things: (1) pulls cross-store ROAS from all three platforms into a unified view, (2) runs Shopify-to-Meta reconciliation to find purchases that happened but weren't attributed, and (3) surfaces budget reallocation recommendations through a HITL (human-in-the-loop) Telegram interface — a human approves every spend change before it executes.

HITL was a deliberate choice. Fully autonomous ad spend changes are too risky when the attribution data itself is unreliable. Better to surface the recommendation and let a human decide, then automate execution of the decision.

**Key decisions:** HITL approval for all spend changes. PostHog for internal analytics. LangGraph for the reasoning loop — same pattern as our trading Oracle.  
**What we learned:** The most valuable output isn't the recommendations — it's the unified attribution view. Just seeing the real numbers across platforms changes how you think about budget allocation.  
**Signal tags:** grow, metaads, amazon, shopify, attribution, roas, hitl, langgraph, ecommerce, adops

---

## Meta → Amazon Attribution Bridge
**Domain:** Grow  
**Repo:** glitch-grow-ads-agent  
**Status:** Shipping  
**Date:** 2026-Q2

We discovered that 60% of Meta ad spend drives Amazon purchases that are completely invisible to the Shopify pixel. A customer sees a Meta ad, clicks through to a product page, then buys on Amazon instead. Meta gets zero credit. The ad looks like it's underperforming. Budget gets cut. The actual driver of Amazon revenue gets starved.

The fix: pipe Amazon Seller Central orders into Meta Conversions API (CAPI) server-side. Match the Amazon purchaser to the Meta ad click using email hash and timestamp. Credit the Meta campaign that drove the sale. Suddenly 40% ROAS looks like 100%+ ROAS on the same spend.

This is the infrastructure piece that makes the ads agent's attribution view accurate. Without it, every ROAS number in the system is systematically understated.

**Key decisions:** Server-side CAPI over browser pixel — more reliable, not blocked by ad blockers. Email hash matching as the attribution signal.  
**What we learned:** The gap between reported ROAS and real ROAS was larger than expected. Most e-commerce operators are flying blind on this.  
**Signal tags:** grow, metaads, amazon, capi, attribution, roas, ecommerce, shopify, invisible-conversions

---

## Social Media Agent — Autonomous Build-in-Public Engine
**Domain:** Grow / Infrastructure  
**Repo:** glitch-social-media-agent  
**Status:** Active  
**Date:** 2026-Q1

The problem: we ship constantly but have no time to post about it. The agent solves this by mining our own GitHub commits and this MILESTONES.md for novel signals, generating short-form video content with Kling 2.0, and publishing across 10+ platforms through a 48-hour Telegram approval gate.

The approval gate was the key design decision. Fully autonomous posting felt too risky — a bad take, an accidentally misleading claim, or a poorly timed post during a market event could do real damage. The agent prepares everything; a human approves before anything goes live. The founder's time cost is under 30 minutes a week.

ORM (online reputation management) runs in parallel. Twitter mentions are classified into 7 tiers by Gemini. Hard-stop phrases (financial guarantees, legal threats) trigger immediate escalation. Normal negative feedback gets a 2-hour review window before auto-response.

Two content paths: AI-generated (from commits) for the brand account, and drive footage (pre-edited clips) for client brands like Namhya Ayurveda.

**Key decisions:** Approval gate over full autonomy. Signed media URLs so the 80MB video files aren't re-uploaded through our server. Content type routing (video / text / image / document) in a single publisher interface.  
**What we learned:** The novelty scorer (LLM-graded, threshold 0.6) filters out routine commits effectively. The real challenge is ensuring the voice prompts produce content that sounds human — not like an AI summarizing a changelog.  
**Signal tags:** grow, social-media, buildinpublic, langgraph, kling, orm, autonomous-agent, content-automation

---

## The Ensemble Pattern
**Domain:** Cross-cutting  
**Status:** Ongoing thesis

Across trade, edge, and grow, we keep applying the same architectural pattern: an ensemble of specialized models, each handling the part of the problem it's best at, coordinated by an Oracle layer that sees the whole picture.

In trading: 9 bots, each with a distinct strategy, coordinated by Oracle risk controls.  
In sports: scenario models + pricing models + lineup models, coordinated by the BettingDecision interface.  
In ad ops: attribution models + recommendation models + execution layer, coordinated by the HITL approval gate.

The pattern generalizes because the underlying problem generalizes: no single model is good at everything, but the right ensemble with the right coordination beats any individual model at its own specialty. The hard part is always the coordination layer, not the individual models.

**Signal tags:** ensemble, ai-architecture, coordination, buildinpublic, glitchexecutor, ailab
