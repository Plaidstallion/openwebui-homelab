# Performance Optimization: Chasing Down a 30-Second Digest

Once the functional pieces of this stack were in place and working reliably — native
function calling, the full Tools ecosystem, the morning digest automation — the next
question was speed. The daily digest (weather + news, two tool calls) was taking
about 30 seconds from pressing enter to the first token appearing on screen. This
doc walks through how that got diagnosed and cut down to roughly half that, with the
real numbers at each step.

## Isolating the problem

The instinct going in was that this could be several different things: Ollama model
load/keep-alive behavior, multi-tool-call round-trip overhead, the SearXNG search
step itself, GPU/VRAM contention with the voice stack's Whisper container, or the
system prompt (grown fairly long after several rounds of steering fixes) adding
per-request overhead.

**First test: cold vs. warm model load.** Ran the same prompt twice, once after
Ollama had been idle a while and once immediately after. Total response time was
about the same either way — this ruled out model loading as the dominant cost
before going any further.

**Getting real numbers.** Ollama's `/api/generate` and `/api/chat` responses (and
what Open-WebUI stores per message) include exact timing fields in nanoseconds:
`load_duration`, `prompt_eval_duration`, `eval_duration`, plus token counts for
each phase. Pulling these from a captured digest run gave the actual breakdown:

| Phase | Time | Share |
|---|---|---|
| Model load | 0.24s | 1% |
| **Prompt eval (prefill)** | **21.0s** | **69%** |
| Generation (output) | 8.5s | 28% |

9,297 input tokens were being processed at only ~443 tokens/sec — unusually slow
for prefill on an RTX 3090, since prefill is normally much faster than
token-by-token generation (it's parallelizable, generation isn't). That pointed at
two separate, stackable problems: something limiting raw prefill throughput, and
the input itself being larger than it needed to be.

## Fix 1: Flash attention was off

Ollama defaults to flash attention disabled. Checked both Machine and User-scope
environment variables on the Windows GPU box and confirmed `OLLAMA_FLASH_ATTENTION`
was unset. Enabled it:

```powershell
# As Administrator:
[System.Environment]::SetEnvironmentVariable("OLLAMA_FLASH_ATTENTION", "1", "Machine")
```

Since Ollama runs as a Task Scheduler task here (see the note on that further down),
the task had to be stopped and restarted to pick up the new environment — a process
launched before the variable is set won't see it.

Result, same ~9,200-token prompt:

| Run | Prompt eval | Prefill speed |
|---|---|---|
| Before | 21.0s | 443 tok/s |
| After | 8.4s | 1,088 tok/s |

A 2.4x prefill speedup from one environment variable, with zero changes to the
prompt or tools.

## Fix 2: The prompt itself was bloated

Separately from prefill *speed*, the 9,297-token input size was itself worth
questioning. Open-WebUI sends the full tool schema for every enabled tool on
every completion call — regardless of which tool actually ends up being used.
This model preset has 11 tools attached (weather, search, crypto, stocks, sports,
exchange rate, flight, transit, traffic, package tracking, image generation), and
six of them (sports, stocks, exchange-rate, flight, transit, traffic — several
still placeholders pending a real data API decision) carried a fairly verbose
docstring:

```
ALWAYS use this tool for ANY question about a stock's price, quote, or how a
company's shares are trading - for example "what's Apple's stock price", "AAPL?",
"what's Tesla trading at right now". Do NOT use web_search for these questions,
even though web_search is technically capable of finding stock prices online:
this tool is the correct, authoritative choice for stock-price queries on this
system, and web_search should be reserved for topics with no dedicated tool.
Calling web_search instead of this tool for a stock-price question is a mistake.
```

That language wasn't arbitrary — it was added earlier in the project specifically
because the model was mis-routing these questions to `web_search` instead of the
dedicated tool, and it was verified fixing that with a repeat-trial test at the
time. So the goal here wasn't to gut the steering, just tighten it:

```
Use for any stock price or quote question (e.g. "AAPL price", "Tesla stock").
Never use web_search for these - this tool is authoritative for stock prices.
```

Same "always use this / never web_search" instruction, a fraction of the tokens.
Applied the same pattern to all six tools, and separately condensed the system
prompt itself (which carries several accumulated fixes — an advice-phrasing
exception, a Knowledge-collection exception, image-markdown handling, gendered-
translation hedging) from 2,919 characters down to 1,599, preserving every rule
but cutting repeated justification text.

Result:

| | Prompt tokens | Prompt eval |
|---|---|---|
| Before trim | ~9,220 | 7.7-11.2s (post flash-attention) |
| After trim | ~8,610-8,660 | 7.7s, consistently |

## Net result

| | Total time | Prompt eval |
|---|---|---|
| Original baseline | ~30s | 21.0s |
| After both fixes | ~15-19s | 7.7s |

Prefill dropped from 21.0s to 7.7s — roughly a 2.7x improvement — split between
flash attention (the bigger win) and the token trim (a smaller but real second
cut). The remaining ~8-11s of total time is generation (writing the actual
response), which scales with output length and is a function of GPU throughput at
this point, not something prompt/tool-schema changes can touch further. `nvidia-smi`
confirmed the 3090 sitting at 100% utilization and ~369-370W during generation —
a genuinely maxed-out consumer GPU, not a bug or a config issue.

One thing that looked like a regression turned out not to be: a run that took
noticeably longer than two runs right before it had *identical* prompt-eval time
(7.7s both) — the difference was entirely down to the model generating more output
tokens that turn (357 vs. 264), not any slowdown in loading or prefill.

## Reproducing this

`OLLAMA_FLASH_ATTENTION=1` is a machine-level environment variable set directly on
the GPU box, outside Docker entirely — it won't show up in any compose file or
Open-WebUI export in this repo, so if you're reproducing this setup, set it
yourself and restart however you're running Ollama (a Task Scheduler task here,
see `docs/wsl2-docker-migration.md` and `docs/image-generation-setup.md` for why
that pattern is used instead of Windows Services on this box).

The trimmed tool docstrings and system prompt are reflected in this repo's
`open-webui/tools/` and `open-webui/models/` exports as of this doc.

## A caution on trimming steering text

After trimming, a regression check (`what's the score of the Packers game?`)
called both `get_sports_score` and `web_search` together — the correct tool still
fired, but so did the redundant one. This might not even be new: the same
double-call pattern was observed and accepted during the original verbose-docstring
testing, before any trimming happened. Worth keeping an eye on if it ever produces
a wrong or slow answer rather than a merely redundant one — if it does, the fix is
adding back a little more explicit steering to that one tool, not reverting
everything.
