# Knowledge / RAG Setup (Open-WebUI)

This uses Open-WebUI's built-in Knowledge feature (document collections) for retrieval-augmented generation — no separate vector DB or extra services required, entirely configured through the browser.

> Note: this section documents the *settings and fixes*, not the documents themselves. Knowledge collections can't be exported without including their underlying documents, so if your collection contains anything private, don't try to export it — just replicate the settings below against your own documents.

## 1. Embedding model

**Admin Panel → Settings → Documents:**
- Embedding Model Engine: `Ollama`
- Embedding Model: `nomic-embed-text`
- Chunk Size: `1000` (adjusted down from an initial 1500 — smaller chunks gave more precise retrieval for shorter, less-dense documents)
- Chunk Overlap: default

`nomic-embed-text` was already pulled via Ollama for other use, so it was reused here rather than adding a dedicated embedding model — one less thing to manage.

No changes were needed on the chat model side — the model already had `num_ctx` set to 16384, comfortably above what's needed for RAG context injection.

## 2. Creating a collection

Straightforward: create a Knowledge collection, upload documents, and it indexes automatically. Attach it to a chat with `#` and ask a question the document actually answers, to confirm retrieval is working end-to-end before trusting it at scale.

Tested at a real scale — 23 documents in one collection — with accurate, citation-backed retrieval across the set.

## 3. The disambiguation bug (and the actual fix)

**The problem:** when a question could plausibly be answered by more than one retrieved document describing *different* procedures for superficially similar situations, the model would silently pick one interpretation and answer confidently — instead of asking which one applied. Retrieval itself was working correctly (the right documents were being pulled), but the model wasn't surfacing the ambiguity.

This is more than a minor inconvenience: in one test case, the model defaulted to the most destructive of three retrieved options (a full account/profile deletion) without mentioning the two safer, less destructive alternatives that were also retrieved.

**First attempt that failed:** adding a disambiguation instruction to the model's system prompt. Tested in both an existing chat and a fresh chat — identical results either way, so it wasn't a stale-context issue. The instruction simply wasn't taking effect.

**Root cause:** Open-WebUI has a separate **RAG Template** (Admin Panel → Settings → Documents), independent from the model's system prompt. Its default phrasing tells the model to answer directly and concisely using only the retrieved context — and because this template sits structurally closer to the actual query than the system prompt does, its instruction was winning regardless of what the system prompt said.

**The actual fix:** edit the RAG Template itself (not the system prompt) to include the disambiguation exception. The default template's task line needs to explicitly allow — and instruct — the model to ask which situation applies when the retrieved context contains multiple genuinely different procedures, especially when they differ in risk or reversibility.

Confirmed fixed on retest: previously-ambiguous questions now correctly pause and ask which situation applies, naming the distinct options, before answering.

**The exact template used (default no longer on hand, so only the fixed version is shown here):**
```
### Task:
Answer the user's question directly and concisely using only the context provided below — except: if the context contains multiple distinct procedures for what could be different underlying problems (not just multiple angles on the same fix), do not pick one and answer directly. Instead, briefly name the different situations the context covers and ask which one applies. Weigh this especially heavily when the procedures differ in risk or reversibility (e.g., one is destructive or hard to undo and another is not) — in that case, name the safer options and ask for confirmation before describing the destructive one.
### Guidelines:
- Do not mention external documents, search results, or the context itself.
- Do not output internal reasoning, thinking, or XML tags.
- If the answer cannot be found in the context, state that you do not have access to real-time information for that query.
<context>
{{CONTEXT}}
</context>
User Question: {{QUERY}}
```

The key addition versus Open-WebUI's stock template is the disambiguation clause in the Task section — everything from "except: if the context contains multiple distinct procedures..." onward. The rest (Guidelines, context/query placeholders) is close to the default structure.

## 4. Known limitation (not fixed, accepted as-is)

Separately from the disambiguation issue: questions that are fully answerable from an attached Knowledge collection sometimes still trigger an unnecessary web search alongside the correct KB retrieval (e.g. an internal-process question also triggering a live search for the same topic).

Attempted fix: adding a "skip web search when a Knowledge collection is attached and the question is internal-process-shaped" exception to the tool-use instructions. This did **not** resolve it on retest.

**Likely cause:** with native function calling, tool selection happens in a single pass, before the model has seen what the Knowledge retrieval actually returns — so it can't conditionally "check the KB first, then decide whether a web search is also needed" the way a written instruction asks it to. This looks like an architectural constraint of native function calling rather than something fixable through prompting.

**Why this was left as-is rather than kept patching:** the extra search is cosmetic and adds latency, but doesn't affect correctness — the final answer still correctly respects the RAG Template and doesn't leak bad information from the unnecessary search. The real fix, if this mattered enough to chase further, would be a dedicated model/workflow where a Knowledge base is always attached by default with no competing web-search tool in the same turn — not something needed here.
