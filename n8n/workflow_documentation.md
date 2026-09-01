# n8n Workflow Documentation

> Workflow file: [`workflow.json`](workflow.json) — import directly into n8n (Cloud or self-hosted).
> Implements the selected use case from [research/use_cases.md](../research/use_cases.md): *fault code + symptom → hardware-vs-connectivity classification → fix guidance or escalation.*
> Trigger/response channel: **Telegram** — chosen over a raw webhook specifically so this is live-demoable during the Round 1 presentation (text the bot, watch it classify and reply in real time) rather than something only visible via curl/Postman.

## Purpose

An installer texts a heat pump manufacturer's support bot on Telegram with a fault code or a plain-language symptom. The workflow classifies it as a **hardware fault**, a **HEMS connectivity issue**, or an **installer error**, then replies with either concrete fix guidance or an escalation instruction — always labeled as AI-suggested triage for a human to confirm, never an autonomous action.

## Trigger(s)

**Telegram Trigger** (`message` update) — fires on every message sent to the bot. Requires a Telegram Bot token from [@BotFather](https://t.me/BotFather), configured as an n8n Telegram credential and attached to both Telegram nodes in the workflow (trigger and the final reply).

## Nodes & steps

| # | Node | Type | What it does |
|---|---|---|---|
| 1 | **Telegram Trigger** | Telegram Trigger | Fires on every incoming chat message. |
| 2 | **Parse Installer Message** | Code | Turns the raw chat message into the same ticket shape the classification logic expects: `chat_id`, `telegram_user`, `fault_code` (extracted via regex if the installer mentions one, e.g. "error code E4"), and `reported_symptom` (the full message text). `model`/`firmware_version`/`installer_type` default to `"not provided via chat"` — see **Limits** below. |
| 3 | **Lookup Known Fault Code** | Code | Checks the extracted `fault_code` against a small deterministic table (`E4`, `E7`, `E12` → hardware_fault; `E2` → installer_error; `CONN-01` → connectivity_issue) mirroring the archetypes in [data/synthetic_fault_dataset.csv](../data/synthetic_fault_dataset.csv). A recognized code is resolved instantly, with no LLM call, no cost, and a fully auditable rule. |
| 4 | **Known Fault Code?** | IF | Branches on whether step 3 found a match. |
| 5 | **Classify via OpenAI** *(false branch only)* | HTTP Request | For messages with no recognized code — the connectivity/installer-error cases in practice, which rarely carry a formal code — sends the symptom text to the OpenAI API (Chat Completions, JSON mode) with a system prompt that classifies into the same 3 categories and returns fix/escalation guidance as JSON. |
| 6 | **Parse OpenAI Response** *(false branch only)* | Code | Parses OpenAI's JSON reply; recovers the original ticket fields (the HTTP node's output replaces `$json`); falls back to a **safe default of escalation** if the model output can't be parsed — it never silently defaults to "no action needed." |
| 7 | **Is Hardware Fault?** | IF | Both branches from step 4 converge here. Routes on the resolved `category`. |
| 8 | **Build Escalation Response** *(true branch)* | Set | Formats an escalation payload: status, category, fault code, source (`lookup`/`llm`), confidence, and a message ending in *"AI-suggested triage, confirm before acting."* Carries `chat_id` forward explicitly so the reply reaches the right chat. |
| 9 | **Build Fix Guidance Response** *(false branch)* | Set | Same shape, but `status: resolved_guidance` and the message leads with *"Suggested fix: ..."* |
| 10 | **Log to Monitoring (LangSmith)** | NoOp (placeholder) | Marks where every triage decision would be posted to LangSmith for tracing in a real deployment — see [langsmith/](../langsmith/) for the actual Round 1 trace sample and monitoring notes. |
| 11 | **Format Telegram Reply** | Code | Turns the structured result into a short chat-formatted reply: a wrench/checkmark emoji, the category in plain language, the message, and the classification source. |
| 12 | **Send Telegram Reply** | Telegram | Sends the formatted reply back to the installer's chat. |

Three sticky notes in the canvas call out the "classify," "route," and "Round 1 scope / Telegram limitation" sections for anyone opening the workflow cold.

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

### Worked examples (traced against real rows in the synthetic dataset, and unit-tested against the actual generated node code)

| Installer message | Path taken | Reply |
|---|---|---|
| "Low refrigerant pressure alarm, error code E4" | `E4` extracted by regex → lookup match → hardware_fault | 🔧 Hardware fault — escalate, source: lookup, confidence 1 |
| "Smart meter gateway will not pair with the control unit" | No code in text → OpenAI classifies → connectivity_issue | ✅ HEMS connectivity issue — re-pair via app, check 2.4GHz band and gateway certificate, source: AI classification |
| "Unit trips on startup, wiring terminal check shows a loose connection" | No code in text → OpenAI classifies → installer_error | ✅ Installer/commissioning error — correct the commissioning step, no escalation, source: AI classification |

The first row runs entirely on the free, deterministic lookup; the other two exercise the OpenAI classification path — together they demonstrate all three categories and both routing branches.

## Environment variables / credentials used

| Credential / variable | Used by | Notes |
|---|---|---|
| Telegram Bot token | Telegram Trigger, Send Telegram Reply | Get one from [@BotFather](https://t.me/BotFather), add as an n8n "Telegram" credential, attach to both Telegram nodes. |
| `OPENAI_API_KEY` | Classify via OpenAI | Referenced as `{{ $env.OPENAI_API_KEY }}` in the HTTP Request node's header (`Authorization: Bearer ...`). Set it as an n8n environment variable, or swap the node for n8n's built-in OpenAI/HTTP Request credential type if preferred. Model is hardcoded to `gpt-4o-mini` with `response_format: json_object` for reliable structured output — edit the `jsonBody` expression to swap models. |

No other credentials are required — the lookup step needs none, and the monitoring step is a placeholder in Round 1 (see Limits below).

## Limits vs. production

- **Telegram only supplies free text.** `model`, `firmware_version`, and `installer_type` default to `"not provided via chat"` because a chat message alone doesn't carry that structured context. A production version would either ask one quick follow-up question, or have the installer app tag the chat with equipment context automatically (e.g. a QR code scanned at the unit) instead of relying on typed text for it.
- **LangSmith is a placeholder node here**, not a live integration — Round 1's actual LangSmith evidence (trace sample, dataset, observability notes) lives in [langsmith/](../langsmith/) as a separate, smaller demonstration, per the capstone brief's "LangSmith monitoring sample" deliverable. Wiring this node to post real traces is Round 2 scope.
- **The fault-code lookup table is intentionally small** (5 codes) — a production version would load it from the manufacturer's actual fault-code database rather than hardcoding it in the Code node.
- **No retry/rate-limit handling** on the OpenAI HTTP Request node, and the bot is only as secure as its Telegram token — both are expected of a Round 1 POC, not a hardened production endpoint.
- **The classifier never issues autonomous repair instructions** for hardware faults by design (see the system prompt in the "Classify via OpenAI" node and [research/opportunities_risks.md](../research/opportunities_risks.md)) — this is a deliberate safety constraint, not a gap to close later.
- **Not run against a live n8n/Telegram bot in this environment** — the JSON was hand-built to n8n's workflow schema; structural correctness (well-formed JSON, all node connections resolve to real node names) was validated automatically, and the two new Code nodes (`Parse Installer Message`, `Format Telegram Reply`) were unit-tested against the exact generated code with the worked examples above. It hasn't, however, been visually confirmed by importing it into n8n and talking to a live bot. Please import `workflow.json`, connect a Telegram bot token, and try the `E4` example first — it needs no OpenAI API key at all, since it's resolved by the lookup step. If anything doesn't import cleanly, it's a normal editable workflow at that point.

## How to run it

1. Message [@BotFather](https://t.me/BotFather) on Telegram, create a bot, copy its token.
2. In n8n: **Import from File** → `workflow.json`.
3. Add a Telegram credential with that token; attach it to **Telegram Trigger** and **Send Telegram Reply** (already referenced by name in the imported JSON — n8n will prompt you to reconnect it).
4. Set `OPENAI_API_KEY` in n8n's environment variables (Settings → Environment, or your n8n host's env config).
5. Activate the workflow.
6. Open a chat with your bot and send one of the worked examples above.
