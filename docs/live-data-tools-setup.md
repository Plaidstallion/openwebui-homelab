# Crypto, Sports & Stock Price Tools

Three more Open-WebUI Native Function Calling Tools, added at the same time as a set: `Crypto_Tool.py` (a real integration), and `Sports_Tool.py` / `Stocks_Tool.py` (honest placeholders, same pattern as `Package_Tracking_Tool.py`).

## Why crypto got a real integration and the other two didn't

All three started as candidates for the placeholder treatment. Crypto ended up different because CoinGecko's API needs no key at all - no signup, no tier decision, nothing blocking a real implementation today. Sports and stocks both have live-data APIs too, but every option comes with a real tradeoff still unresolved:

- **Sports**: ESPN's unofficial API (no key, undocumented, could break without notice), TheSportsDB (free tier, needs a key), or SportRadar/API-Sports paid tiers. A free API-Sports attempt was already tried and reverted for NFL/Premier League/Bundesliga - the free tier restricted current-season data and had unreliable team-name matching.
- **Stocks**: Yahoo Finance's unofficial endpoints (no key, same fragility profile as ESPN's) or a keyed free tier like Alpha Vantage/Finnhub.

Rather than build another half-working integration, both stayed as placeholders until one of those API/tier decisions actually gets made.

## Crypto_Tool.py

`get_crypto_price(coin, vs_currency="")` resolves whatever the model passes in - a full name or a ticker symbol - via CoinGecko's `/search` endpoint, preferring an exact name/symbol match and falling back to CoinGecko's top-ranked result otherwise. It then pulls price, 24h change, market cap, and 24h volume from `/simple/price`.

Real-world tested and confirmed working: full-name and ticker lookups, non-default currencies, casual/advice-style phrasing, German phrasing, and a deliberately tricky naming-collision case - "Terra" correctly resolved to the new Terra 2.0 chain (LUNA) rather than Terra Classic (LUNC), matching live CoinGecko data. The outage/network-failure fallback path (an honest error message pointing to CoinGecko directly) hasn't been tested against a real outage yet, just reviewed in code.

## Sports_Tool.py / Stocks_Tool.py

Both follow the same shape: a single method (`get_sports_score` / `get_stock_price`) that never fabricates a score or price - it explains that live data isn't integrated yet, briefly why, and points to a fallback URL (ESPN, Yahoo Finance) for a manual check in the meantime.

## The bug: web_search was winning over the placeholders

Both placeholder tools were attached and enabled the whole time (confirmed in the model's Tools panel, Function Calling set to Native), but the very first round of testing showed `web_search` firing instead of the new tools on direct sports/stock questions - "What's the score of the Packers game?" only ever showed `View Result from web_search`, never `get_sports_score`.

This wasn't the tools being unreachable; it was tool *selection* - `web_search` is general-purpose enough to plausibly answer the same questions, and the model was reaching for the tool it already knew over the new, narrower one.

**First fix attempt:** rewrote both docstrings with explicit steering language - "ALWAYS use this tool for X... do NOT use web_search... calling web_search instead is a mistake" - the same kind of directive phrasing that fixed an earlier advice-phrasing bug on `Search_Tool.py`. Retesting the exact same Packers prompt afterward still showed `web_search` alone, no placeholder - so the docstring change alone didn't resolve it on that retest.

**What actually happened on repeat testing:** running the same prompt again in a fresh session, and then a batch of 5 varied prompts (single-team scores, stock price via two phrasings, a combined sports+crypto multi-tool query), the model consistently called **both** the placeholder tool and `web_search` together every time - the placeholder's honest "not built yet" message showed up in the tool-result panel, and `web_search` supplied the real answer for the user-facing response. The one earlier miss (web_search alone) is treated as one-off flakiness rather than a persistent bug, consistent with a similar one-off `web_search`-only miss seen earlier in this project that also self-resolved without any code change.

**Net result:** considered resolved. The model reliably surfaces the placeholder's honest disclosure while still giving the user a real, useful answer via `web_search` - arguably a better outcome than a placeholder alone, since the user isn't left without an answer while a real sports/stocks integration is still pending.
