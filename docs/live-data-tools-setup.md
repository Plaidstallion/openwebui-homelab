# Crypto, FX, Sports, Stocks, Transit, Flight & Traffic Tools

Seven Open-WebUI Native Function Calling Tools covering live/real-time data: `Crypto_Tool.py` and `FX_Tool.py` are real integrations against free, keyless APIs. `Sports_Tool.py`, `Stocks_Tool.py`, `Transit_Tool.py`, `Flight_Tool.py`, and `Traffic_Tool.py` are honest placeholders, same pattern as `Package_Tracking_Tool.py` - they never fabricate data, they explain that real-time lookups aren't wired up yet and point to a fallback URL.

## Why some got real integrations and others didn't

The deciding factor was simple: does a genuinely free, keyless API exist? Crypto (CoinGecko) and FX (Frankfurter) both do, so they got real implementations. Every other candidate has a real live-data API available, but each comes with an unresolved tradeoff:

- **Sports**: ESPN's unofficial API (no key, undocumented, could break without notice), TheSportsDB (free tier, needs a key), or SportRadar/API-Sports paid tiers. A free API-Sports attempt was already tried and reverted for NFL/Premier League/Bundesliga - the free tier restricted current-season data and had unreliable team-name matching.
- **Stocks**: Yahoo Finance's unofficial endpoints (no key, same fragility profile as ESPN's) or a keyed free tier like Alpha Vantage/Finnhub.
- **Transit**: no general-purpose keyless option; would need a regional GTFS real-time feed or a service like Transit App/Google Maps Transit.
- **Flights**: AeroDataBox, FlightAware's AeroAPI, or similar - all keyed/paid tiers.
- **Traffic**: Google Maps' Roads/Directions API, TomTom, or similar - all keyed/paid tiers.

News headlines were considered and dropped entirely - `web_search`/SearXNG already handles news queries reliably (see the Bundestagswahl date-awareness fix in the main filter docs), so a dedicated news tool would just be redundant.

## Crypto_Tool.py

`get_crypto_price(coin, vs_currency="")` resolves whatever the model passes in - a full name or a ticker symbol - via CoinGecko's `/search` endpoint, preferring an exact name/symbol match and falling back to CoinGecko's top-ranked result otherwise. It then pulls price, 24h change, market cap, and 24h volume from `/simple/price`.

Real-world tested and confirmed working: full-name and ticker lookups, non-default currencies, casual/advice-style phrasing, German phrasing, and a deliberately tricky naming-collision case - "Terra" correctly resolved to the new Terra 2.0 chain (LUNA) rather than Terra Classic (LUNC), matching live CoinGecko data. The outage/network-failure fallback path (an honest error message pointing to CoinGecko directly) hasn't been tested against a real outage yet, just reviewed in code.

## FX_Tool.py

`get_exchange_rate(from_currency, to_currency, amount=1.0)` maps common currency names ("dollars", "euros", "yen", ...) to ISO codes, then calls Frankfurter's `https://api.frankfurter.dev/v2/rate/{base}/{quote}` - free, no key, no quota. Real-world tested and confirmed working: ISO codes, common-name aliases (including multi-word ones like "Canadian dollars"), non-default amounts, invalid currency codes (honest 404-style message), advice-phrasing, German phrasing, and multi-tool combos with both Crypto_Tool.py and Weather_Tool.py.

**A labeling bug worth knowing about:** the tool's output originally described the data as "ECB daily reference rate," but Frankfurter's v2 API actually blends up to 84 central banks by default - not ECB-only. This surfaced when a Saturday-night test showed a same-day date, which shouldn't happen for ECB-only data (the ECB doesn't publish on weekends). Confirmed directly: pinning to `?providers=ECB` correctly showed the prior business day (Friday), while the blended default showed the same-day (Saturday) date - proving the blend includes providers with different publishing calendars. Fixed by changing the in-response caveat to say "blended daily reference rate from central bank sources" instead of naming ECB specifically. The tool doesn't currently expose a `providers` parameter for pinning to a single source (e.g. strict ECB) - left as a possible future addition, not needed for the current use case.

## Sports_Tool.py / Stocks_Tool.py / Transit_Tool.py / Flight_Tool.py / Traffic_Tool.py

All five follow the same shape: a single method that never fabricates a result - it explains that live data isn't integrated yet, briefly why, and points to a fallback URL (ESPN, Yahoo Finance, Google Maps Transit, FlightAware, Google Maps traffic layer) for a manual check in the meantime.

## The bug: web_search was winning over the placeholders (Sports/Stocks only)

Sports_Tool.py and Stocks_Tool.py were attached and enabled the whole time (confirmed in the model's Tools panel, Function Calling set to Native), but the very first round of testing showed `web_search` firing instead of the new tools on direct sports/stock questions - "What's the score of the Packers game?" only ever showed `View Result from web_search`, never `get_sports_score`.

This wasn't the tools being unreachable; it was tool *selection* - `web_search` is general-purpose enough to plausibly answer the same questions, and the model was reaching for the tool it already knew over the new, narrower ones.

**First fix attempt:** rewrote both docstrings with explicit steering language - "ALWAYS use this tool for X... do NOT use web_search... calling web_search instead is a mistake" - the same kind of directive phrasing that fixed an earlier advice-phrasing bug on `Search_Tool.py`. Retesting the exact same Packers prompt afterward still showed `web_search` alone, no placeholder - so the docstring change alone didn't resolve it on that retest.

**What actually happened on repeat testing:** running the same prompt again in a fresh session, and then a batch of 5 varied prompts (single-team scores, stock price via two phrasings, a combined sports+crypto multi-tool query), the model consistently called **both** the placeholder tool and `web_search` together every time - the placeholder's honest "not built yet" message showed up in the tool-result panel, and `web_search` supplied the real answer for the user-facing response. The one earlier miss (web_search alone) is treated as one-off flakiness rather than a persistent bug, consistent with a similar one-off `web_search`-only miss seen elsewhere in this project that also self-resolved without any code change.

**Net result:** considered resolved. The model reliably surfaces the placeholder's honest disclosure while still giving the user a real, useful answer via `web_search` - arguably a better outcome than a placeholder alone, since the user isn't left without an answer while a real sports/stocks integration is still pending.

Given this history, Transit_Tool.py, Flight_Tool.py, Traffic_Tool.py, and FX_Tool.py all shipped with the steering language built into v1.0 from the start rather than retrofitted. Transit and Flight were real-world tested (German advice-phrasing and vague "how's my flight looking" both correctly triggered the tool and asked for missing details rather than guessing or falling back to `web_search`; a meta-phrasing question like "what tools do you have for transit" correctly did *not* trigger the tool, same accepted edge case as package tracking). Traffic_Tool.py has not yet been individually stress-tested beyond a quick spot check, but given how consistently the same pattern held for Transit and Flight, no issues are expected.
