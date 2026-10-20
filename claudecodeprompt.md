# Claude Code Prompt, Apply xbot Algorithm Fixes

> **Instructions:** Copy everything below this line and paste it as your prompt to Claude Code, run it from the root of `/xbot`.

---

```

This is an autonomous X (Twitter) bot called xbot that posts cryptocurrency market data to @TitanTechIn. The full algorithmic audit report is available at: https://github.com/entropiacity/titanxbot

Your task is to apply the following 5 targeted improvements to the codebase. Read each file before editing. Preserve all existing docstrings, comments, and logic that is unrelated to the changes below. Run the test suite after all changes and confirm 184 tests pass.

---

## CHANGE 1, Cashtag Auto-Injection (src/xbot/persona.py)

Add the following at the top of `src/xbot/persona.py`, after the existing imports:

```python
import re

_CASHTAG_MAP = {
    "btcusdt": "$BTC", "btc": "$BTC",
    "ethusdt": "$ETH", "eth": "$ETH",
    "solusdt": "$SOL", "sol": "$SOL",
    "xrpusdt": "$XRP", "xrp": "$XRP",
    "bnbusdt": "$BNB", "bnb": "$BNB",
}
_CASHTAG_RE = re.compile(
    r"\b(" + "|".join(_CASHTAG_MAP.keys()) + r")\b",
    re.IGNORECASE,
)

def inject_cashtags(text: str) -> str:
    """Replace bare ticker symbols with X cashtag format for search indexing.
    
    Converts BTCUSDT → $BTC, ETHUSDT → $ETH, etc. so posts are indexed
    in X's financial search streams and reach non-followers via SimClusters.
    """
    return _CASHTAG_RE.sub(lambda m: _CASHTAG_MAP[m.group(1).lower()], text)
```

Then find the function or method that finalizes tweet text before posting (likely in `action/engine.py` or called from there) and add `text = inject_cashtags(text)` as the last transformation step before the text is sent to the Tweepy client.

---

## CHANGE 2, Conversational Hook in SYSTEM_PERSONA (src/xbot/persona.py)

Find the `SYSTEM_PERSONA` string (around lines 85–120). Locate the "substance rules" or "format rules" section.

Add these two rules to that section, preserving all existing rules:

```
- Always end market takes with a sharp, open-ended trade tension question that 
  invites active traders to debate (e.g., "local squeeze or range rejection?", 
  "who's stepping in to bid this dip?", "is this a fakeout or a confirmed break?").
  This question must reflect a genuine uncertainty in the data, not a rhetorical statement.
- Every post referencing BTC, ETH, or SOL must use the $CASHTAG format ($BTC, $ETH, $SOL)
  to enable X's financial search indexing.
```

---

## CHANGE 3, Active Presence Window (src/xbot/decision/arbiter.py)

Read `src/xbot/decision/arbiter.py` fully. 

If a field named `ACTIVE_HOURS_UTC` or `active_hours_utc` does not already exist in the settings/config class, add it:

```python
ACTIVE_HOURS_UTC: list[int] = Field(default=[7, 21], alias="ACTIVE_HOURS_UTC")
```

Then find or add a method `_in_presence_window` on the arbiter class:

```python
def _in_presence_window(self, now: datetime) -> bool:
    """Return True only if current UTC hour is within the operator's active window.
    
    Non-critical posts should only fire when the operator is online and can
    respond to replies within 30 minutes, capturing the +75.0 author-reply
    HeavyRanker multiplier in X's recommendation algorithm.
    """
    start, end = self._s.active_hours_utc
    return start <= now.hour < end
```

Wire this check into the post-scheduling logic so that routine posts (non-critical market takes) only fire during the active window. Critical posts (liquidations > $25M, major level breaks) should bypass this check.

---

## CHANGE 4, Fix Quote-Tweet 403 Error (src/xbot/config.py)

Open `src/xbot/config.py`. Find line 249 (approximately) where `comment_approval` is defined.

Change:
```python
comment_approval: bool = Field(False, alias="COMMENT_APPROVAL")
```
To:
```python
comment_approval: bool = Field(True, alias="COMMENT_APPROVAL")
```

This prevents the commenter module from automatically firing quote-tweets that the Basic-tier X API rejects with 403 Forbidden.

---

## CHANGE 5, Post-Drop Operator Webhook Alert (src/xbot/action/engine.py)

Read `src/xbot/action/engine.py` fully. Find the section where a successful tweet post is confirmed (where `tweet_id` or the tweet URL is available after a successful Tweepy API call).

After the successful post confirmation, add a call to notify the operator. If a `notify_operator` function or Telegram/webhook notifier already exists in the codebase, use it. Otherwise, create a stub:

```python
async def _notify_operator_post_live(tweet_id: str, text_preview: str) -> None:
    """Send an operator alert the moment a tweet goes live.
    
    The operator must respond to any early replies within 30 minutes to
    capture the +75.0 author-reply HeavyRanker multiplier, the single
    highest-weighted positive signal in X's recommendation algorithm.
    
    Wire this to Telegram, Discord, or any webhook via OPERATOR_WEBHOOK_URL
    in the .env config.
    """
    url = f"https://x.com/TitanTechIn/status/{tweet_id}"
    message = (
        f"🚀 Live: {url}\n"
        f"Preview: {text_preview[:120]}...\n\n"
        f"⚡ Reply to any comments NOW to capture the +75x algorithm multiplier."
    )
    webhook_url = os.getenv("OPERATOR_WEBHOOK_URL")
    if webhook_url:
        try:
            import httpx
            async with httpx.AsyncClient() as client:
                await client.post(webhook_url, json={"text": message}, timeout=5)
        except Exception:
            pass  # Non-blocking; never crash the post loop on a notification failure
```

Call this function immediately after a successful post, passing the tweet ID and the first 120 characters of the post text.

---

## CHANGE 6, MongoDB Connection Timeout (src/xbot/db/client.py)

Open `src/xbot/db/client.py`. Find the `Database.connect()` method (around line 38).

Add timeout parameters to the MongoClient constructor if they are not already present:

```python
# Add these kwargs to the MongoClient(...) call:
serverSelectionTimeoutMS=5000,
connectTimeoutMS=5000,
socketTimeoutMS=10000,
```

This prevents the script from hanging for 30+ seconds when run outside the EC2 IP whitelist.

---

## AFTER ALL CHANGES

1. Run `.venv/bin/pytest -v` and confirm all 184 tests pass (2 deprecation warnings for `datetime.utcnow()` are acceptable).
2. Run a quick import check: `PYTHONPATH=src .venv/bin/python -c "from xbot.persona import inject_cashtags; print(inject_cashtags('BTCUSDT rose to 80000'))"`, expected output: `$BTC rose to 80000`.
3. Report a summary of every file modified and every line changed.
```
```
