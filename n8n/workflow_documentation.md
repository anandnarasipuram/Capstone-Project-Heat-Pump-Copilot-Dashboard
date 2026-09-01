# n8n Workflow Documentation

> Workflow file: [`workflow.json`](workflow.json) — import directly into n8n (Cloud or self-hosted).
> Implements the selected use case from [research/use_cases.md](../research/use_cases.md): *fault code + symptom → hardware-vs-connectivity classification → fix guidance or escalation.*
> Trigger/response channel: **Telegram** — chosen over a raw webhook specifically so this is live-demoable during the Round 1 presentation (text the bot, watch it classify and reply in real time) rather than something only visible via curl/Postman.

## Purpose

An installer texts a heat pump manufacturer's support bot on Telegram with a fault code or a plain-language symptom. The workflow classifies it as a **hardware fault**, a **HEMS connectivity issue**, or an **installer error**, then replies with either concrete fix guidance or an escalation instruction — always labeled as AI-suggested triage for a human to confirm, never an autonomous action.

As of this revision, both the deterministic lookup and the LLM classification path are grounded in a small **manual knowledge base** ([data/manuals/](../data/manuals/)) built from real Vaillant aroTHERM and Octopus Cosy documentation, so fix guidance reflects documented manufacturer behavior rather than the model's generic HVAC knowledge alone — see **Manual grounding** below.

## Trigger(s)

**Telegram Trigger** (`message` update) — fires on every message sent to the bot. Requires a Telegram Bot token from [@BotFather](https://t.me/BotFather), configured as an n8n Telegram credential and attached to both Telegram nodes in the workflow (trigger and the final reply).

## Nodes & steps

| # | Node | Type | What it does |
|---|---|---|---|
| 1 | **Telegram Trigger** | Telegram Trigger | Fires on every incoming chat message. |
| 2 | **Parse Installer Message** | Code | Turns the raw chat message into the same ticket shape the classification logic expects: `chat_id`, `telegram_user`, `fault_code` (extracted via regex — synthetic codes like `E4`/`CONN-01`, or real Vaillant-style codes like `F.22`/`F22`/`F9998`, normalized to `F.NN`), and `reported_symptom` (the full message text). `model`/`firmware_version`/`installer_type` default to `"not provided via chat"` — see **Limits** below. |
| 3 | **Lookup Known Fault Code** | Code | Checks the extracted `fault_code` against a deterministic table: the original 5 synthetic codes (mirroring [data/synthetic_fault_dataset.csv](../data/synthetic_fault_dataset.csv)) plus 8 real Vaillant fault codes paraphrased from [data/manuals/fault_code_knowledge_base.json](../data/manuals/fault_code_knowledge_base.json). A recognized code is resolved instantly, with no LLM call, no cost, and a fully auditable rule, and now also carries a `manual_sources` citation for the real codes. |
| 4 | **Known Fault Code?** | IF | Branches on whether step 3 found a match. |
| 5 | **Retrieve Manual Context** *(false branch only)* | Code | For symptoms with no recognized code, does a lightweight keyword match against the manual knowledge base ([data/manuals/connectivity_status_guide.json](../data/manuals/connectivity_status_guide.json), [safety_device_reference.json](../data/manuals/safety_device_reference.json)) and attaches up to 2 matching excerpts as `manual_context` (plus a `manual_sources` citation list) for the LLM step to ground its answer in. |
| 6 | **Classify via OpenAI** | HTTP Request | Sends the symptom text — plus any `manual_context` from step 5 — to the OpenAI API (Chat Completions, JSON mode). The system prompt instructs the model to treat manual excerpts as authoritative when present, and to reply in whichever language the installer used. |
| 7 | **Parse OpenAI Response** | Code | Parses OpenAI's JSON reply; recovers the original ticket fields — including `manual_context`/`manual_sources` — from **Retrieve Manual Context** (not the HTTP node's own output, which replaces `$json`); falls back to a **safe default of escalation** if the model output can't be parsed — it never silently defaults to "no action needed." |
| 8 | **Is Hardware Fault?** | IF | Both branches converge here. Routes on the resolved `category`. |
| 9 | **Build Escalation Response** *(true branch)* | Set | Formats an escalation payload: status, category, fault code, source (`lookup`/`llm`), confidence, `manual_sources`, and a message ending in *"AI-suggested triage, confirm before acting."* — with *"(Grounded in: ...)"* appended when a manual source was used. Carries `chat_id` forward explicitly so the reply reaches the right chat. |
| 10 | **Build Fix Guidance Response** *(false branch)* | Set | Same shape, but `status: resolved_guidance` and the message leads with *"Suggested fix: ..."* |
| 11 | **Log to Monitoring (LangSmith)** | NoOp (placeholder) | Marks where every triage decision would be posted to LangSmith for tracing in a real deployment — see [langsmith/](../langsmith/) for the actual Round 1 trace sample and monitoring notes. |
| 12 | **Format Telegram Reply** | Code | Turns the structured result into a short chat-formatted reply: a wrench/checkmark emoji, the category in plain language, the message, and a source footer that distinguishes "fault-code lookup" / "AI classification" / "AI classification, grounded in manufacturer manual". |
| 13 | **Send Telegram Reply** | Telegram | Sends the formatted reply back to the installer's chat. |

Four sticky notes in the canvas call out the "classify" (including manual grounding), "route," "Round 1 scope," and "language handling" sections for anyone opening the workflow cold.

## Manual grounding

The classification prompt used to rely entirely on the model's own general HVAC knowledge. It's now grounded in a small, sourced knowledge base — see [data/manuals/README.md](../data/manuals/README.md) for full source citations and copyright notes:

- **Deterministic lookup** gained 8 real Vaillant aroTHERM fault codes (e.g. `F.532`, `F.752`, `F.9998`) alongside the original 5 synthetic ones — several of which Vaillant's own documentation flags as *installer-first* issues (wiring, electrical supply, flow balancing) rather than hardware faults, which is a genuine, sourced example of the exact hardware-vs-installer-error ambiguity this project exists to resolve.
- **Free-text symptoms** get a keyword-retrieval pass (`Retrieve Manual Context`) against a small corpus built from the Octopus Cosy Hub/Pod status-light guide (for connectivity_issue) and the aroTHERM manual's safety-device section (for hardware_fault / installer_error). Matches are injected into the OpenAI prompt as "manual excerpts," and the model is instructed to prefer them over its own generic assumptions.
- This is **keyword-based (lexical) retrieval, not embeddings-based** — a deliberate Round 1 simplification that needs no vector store or extra API calls. `cost_estimation/cost_analysis.md` already budgets for the embeddings/vector-store upgrade as Round 2 scope.
- The manuals used are **UK Vaillant/Octopus documentation, standing in as representative content** for Round 1 — not Chleo's own product manuals. Content is paraphrased into structured entries, not reproduced verbatim, for both copyright and practical (structured-lookup) reasons. The fault codes were spot-checked against independent installer references during Round 1 — see [data/manuals/README.md](../data/manuals/README.md) for exactly which codes were verified.

## Language

German installers are the primary real-world user for this pilot, but the manual corpus and this documentation are in English. Two different mechanisms handle this, at two different strengths:

1. **The LLM classification step handles language natively.** The system prompt instructs the model to reply in whichever language the installer used — a German symptom gets German fix guidance, with no separate translation call needed. This is the strong, reliable part of the current design.
2. **The keyword retrieval step is weaker.** It's plain English substring matching, so a handful of German synonyms (`verbindung`, `netzwerk`, `wasserdurchfluss`, `kommunikationsfehler`, etc.) are mixed into each entry's keyword list as a lightweight bridge. This only covers the specific German words already anticipated — a differently-phrased German symptom (a different compound word, a synonym, a typo) simply won't match, and the LLM then classifies from the symptom text alone with no manual grounding, silently falling back to its own general knowledge rather than failing loudly. That's a real, stated limitation, not a hidden one.

**Recommendation:** keep language mechanism (1) as-is — it's already doing the right thing at zero extra cost. Don't try to patch mechanism (2) with more hand-maintained keyword translations; that approach doesn't scale past a handful of terms per language. The actual fix is the Round 2 upgrade already on the roadmap: swap the keyword match for **multilingual embeddings** (e.g. OpenAI's `text-embedding-3` family, which handles German and English in the same vector space) — a German query then retrieves the right English-language manual excerpt on semantic similarity, not exact keyword overlap, and the keyword-translation problem disappears entirely rather than needing to be maintained. Until that upgrade lands, it's worth logging cases where `manual_context` comes back empty for a German-language message, specifically to see how often real installer phrasing falls outside the hand-picked keyword list — that log is exactly the evidence needed to justify (or size) the Round 2 embeddings work.

## Inputs / outputs

**Input:** whatever the installer types into the Telegram chat, e.g.:
```
Low refrigerant pressure alarm, error code E4
```

**Output (Telegram reply):**
```
🔧 *Hardware fault*

Escalation required: Escalate to hardware service technician (compressor/refrigerant circuit inspection). Not resolvable by app/firmware update. — AI-suggested triage, confirm before acting.

_Source: fault-code lookup (confidence 1)_
```

### Worked examples

All five rows below were unit-tested directly against the actual generated node code (`Parse Installer Message` → `Lookup Known Fault Code` → `Retrieve Manual Context`), confirming the regex extraction, code normalization, and keyword scoring behave exactly as shown — see **Limits** for what that testing does and doesn't cover (it doesn't call the real OpenAI API, so the LLM-classified rows show the *grounding* that reaches the model, not a live model response).

| Installer message | Path taken | Reply |
|---|---|---|
| "Low refrigerant pressure alarm, error code E4" | `E4` extracted by regex → lookup match → hardware_fault | 🔧 Hardware fault — escalate, source: fault-code lookup, confidence 1 |
| "No comms from the outdoor unit, controller not responding" | No code in text → keyword match on "controller not responding" → OpenAI classifies, grounded in the F.9998 excerpt → installer_error | ✅ Installer/commissioning error — check eBUS wiring (cable type, polarity, clearance from power lines) before escalating — *(Grounded in: Vaillant aroTHERM fault code reference (F.9998))* |
| "Fault code F532 showing, low flow rate" | `F532` normalized to `F.532` → lookup match → installer_error | ✅ Installer/commissioning error — check building circuit for blockage/air/balancing before escalating, source: fault-code lookup |
| "Wasserdurchfluss Problem, Pumpe steht" (German) | No code → keyword match on "wasserdurchfluss" → OpenAI classifies **and replies in German**, grounded in the F.532/F.788 excerpt → installer_error | ✅ (German-language fix guidance) — *(Grounded in: Vaillant aroTHERM fault code reference (F.532 / F.788))* |
| "Smart meter gateway will not pair with the control unit" | No code, no keyword match in the manual corpus → OpenAI classifies from symptom text alone (no grounding) → connectivity_issue | ✅ HEMS connectivity issue — re-pair via app, check 2.4GHz band and gateway certificate, source: AI classification |

## Environment variables / credentials used

| Credential / variable | Used by | Notes |
|---|---|---|
| Telegram Bot token | Telegram Trigger, Send Telegram Reply | Get one from [@BotFather](https://t.me/BotFather), add as an n8n "Telegram" credential, attach to both Telegram nodes. |
| `OPENAI_API_KEY` | Classify via OpenAI | Referenced as `{{ $env.OPENAI_API_KEY }}` in the HTTP Request node's header (`Authorization: Bearer ...`). Set it as an n8n environment variable, or swap the node for n8n's built-in OpenAI/HTTP Request credential type if preferred. Model is hardcoded to `gpt-4o-mini` with `response_format: json_object` for reliable structured output — edit the `jsonBody` expression to swap models. |

No other credentials are required — the lookup and manual-retrieval steps need none (no vector store, no embeddings API), and the monitoring step is a placeholder in Round 1 (see Limits below).

## Limits vs. production

- **Telegram only supplies free text.** `model`, `firmware_version`, and `installer_type` default to `"not provided via chat"` because a chat message alone doesn't carry that structured context. A production version would either ask one quick follow-up question, or have the installer app tag the chat with equipment context automatically (e.g. a QR code scanned at the unit) instead of relying on typed text for it.
- **Manual grounding is keyword-based and English-only under the hood** (see **Language** above) — a real deployment should upgrade to multilingual embeddings once German-language coverage gaps are logged and quantified.
- **The manual corpus is small and drawn from representative UK manuals, not Chleo's own product line** (see [data/manuals/README.md](../data/manuals/README.md)) — a real deployment ingests the manufacturer's actual documentation.
- **LangSmith is a placeholder node here**, not a live integration — Round 1's actual LangSmith evidence (trace sample, dataset, observability notes) lives in [langsmith/](../langsmith/) as a separate, smaller demonstration, per the capstone brief's "LangSmith monitoring sample" deliverable. Wiring this node to post real traces is Round 2 scope.
- **The fault-code lookup table is still small** (13 codes total now, up from 5) — a production version would load it from the manufacturer's actual fault-code database rather than hardcoding it in the Code node.
- **No retry/rate-limit handling** on the OpenAI HTTP Request node, and the bot is only as secure as its Telegram token — both are expected of a Round 1 POC, not a hardened production endpoint.
- **The classifier never issues autonomous repair instructions** for hardware faults by design (see the system prompt in the "Classify via OpenAI" node and [research/opportunities_risks.md](../research/opportunities_risks.md)) — this is a deliberate safety constraint, not a gap to close later.
- **Not run against a live n8n/Telegram bot in this environment** — the JSON was hand-built to n8n's workflow schema; structural correctness (well-formed JSON, all 17 node connections resolve to real node names, no orphaned nodes) was validated automatically, and every Code node (`Parse Installer Message`, `Lookup Known Fault Code`, `Retrieve Manual Context`, `Parse OpenAI Response`, `Format Telegram Reply`) was unit-tested against the exact generated code with the worked examples above, including the German-language and manual-grounding cases. It hasn't, however, been visually confirmed by importing it into n8n and talking to a live bot — that step still needs doing, the same as the workflow's previous revision. Please import `workflow.json`, connect a Telegram bot token, and try the `E4` example first — it needs no OpenAI API key at all, since it's resolved by the lookup step. If anything doesn't import cleanly, it's a normal editable workflow at that point.

## How to run it

1. Message [@BotFather](https://t.me/BotFather) on Telegram, create a bot, copy its token.
2. In n8n: **Import from File** → `workflow.json`.
3. Add a Telegram credential with that token; attach it to **Telegram Trigger** and **Send Telegram Reply** (already referenced by name in the imported JSON — n8n will prompt you to reconnect it).
4. Set `OPENAI_API_KEY` in n8n's environment variables (Settings → Environment, or your n8n host's env config).
5. Activate the workflow.
6. Open a chat with your bot and try the worked examples above — start with `E4` (needs no API key at all), then try one of the free-text ones to see the manual grounding and citation in action, and the German example to see the language handling.
