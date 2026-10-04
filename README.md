<div align="center">

# Conversational BI for Power BI — built entirely on Power Platform

**Ask your semantic model a question in plain language. Get DAX, data, and an answer you can verify.**
No Python. No server. No app registration.

[![Power Automate](https://img.shields.io/badge/Power%20Automate-flow-0066FF?logo=powerautomate&logoColor=white)](https://make.powerautomate.com)
[![Power Apps](https://img.shields.io/badge/Power%20Apps-canvas%20app-742774?logo=powerapps&logoColor=white)](https://make.powerapps.com)
[![Power BI](https://img.shields.io/badge/Power%20BI-executeQueries-F2C811?logo=powerbi&logoColor=black)](https://learn.microsoft.com/en-us/rest/api/power-bi/datasets/execute-queries)
[![Claude](https://img.shields.io/badge/Claude-API-D97757)](https://platform.claude.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

</div>

---

> **Text-to-SQL is a gamble. Text-to-DAX on a well-built semantic model is not.**
>
> The difference is where business logic lives. In a raw database the model has to guess how
> margin is calculated, which rows to exclude, how returns are handled. In a Power BI semantic
> model that knowledge is already encoded in the measures. The LLM does not invent business
> logic — it picks the right measure and the right dimension. **Your data model becomes the
> guardrail.**

A question in natural language goes in, a validated DAX query runs against the published
semantic model, and a short analytical answer comes back — with the generated query shown next
to it, so nobody has to take the number on faith.

Everything runs inside tools most organisations already have: **one Power Automate flow and one
Power Apps canvas app**.

---

## Architecture

```
  Power Apps (canvas)
         │  question
         ▼
  ┌──────────────────────────── Power Automate ────────────────────────────┐
  │                                                                        │
  │   SCHEMAT ─┐                                                           │
  │   SYSTEM ──┼─► PROMPT ─► PROMPT_JSON ─► HTTP: Claude  ──► varDax        │
  │   PYTANIE ─┘                            (text → DAX)      │            │
  │                                                           ▼            │
  │                                              ┌─ validation: EVALUATE?  │
  │                                       false ─┤                         │
  │                                              └─ true ─► Power BI       │
  │                                                         executeQueries │
  │                                                              │         │
  │                        HTTP: Claude ◄── PROMPT2_JSON ◄───────┘         │
  │                        (rows → answer)                                 │
  └────────────────────────────────┬───────────────────────────────────────┘
                                   ▼
              answer  ·  generated DAX  ·  raw rows  →  Power Apps
```

Eleven actions. The Power BI connector is **standard** — the only premium component is the
HTTP action that calls the LLM.

---

## What it does

| | |
|---|---|
| **Semantic-model aware** | Tables, columns, measures with their DAX expressions and relationships are injected into the prompt |
| **Transparent** | Every answer ships with the generated DAX — reviewable, copyable, auditable |
| **Guarded** | A condition blocks anything that is not `EVALUATE`/`DEFINE` from reaching Power BI |
| **Honest** | Out-of-scope questions return the model's explanation, not an invented number |
| **Self-healing** *(optional)* | Engine errors are fed back to the model, which rewrites the query |
| **Authentication for free** | The Power BI connector uses the signed-in user's identity — RLS works out of the box |

---

## Repository contents

```
├── docs/
│   ├── POWER_APPS_KROK_PO_KROKU.md   # full click-by-click build guide (PL)
│   └── TESTY.md                      # 25-question evaluation set (PL)
├── flow/
│   ├── system-prompt.txt             # system prompt: question → DAX
│   ├── system-prompt-answer.txt      # system prompt: rows → answer
│   ├── http-dax-request.json         # HTTP body, structured JSON output
│   ├── http-answer-request.json      # HTTP body, natural-language answer
│   └── expressions.md                # every Power Automate / Power Fx expression
└── schema.example.txt                # schema format pasted into the SCHEMAT action
```

The build guide is in Polish; the prompts, expressions and this README are in English.
It covers every action, every expression, and the failure modes behind each design decision —
including the ones listed under *Things that bite* below.

---

## Build it

Full walkthrough in **[`docs/POWER_APPS_KROK_PO_KROKU.md`](docs/POWER_APPS_KROK_PO_KROKU.md)**.
The short version:

### Prerequisites

- An environment with **premium connectors** — Power Apps Developer Plan (free, no expiry) or a trial
- **Build** permission on the semantic model
- Admin portal → Tenant settings → Integration settings → **Dataset Execute Queries REST API = Enabled**
- A Claude API key with credits ([console.anthropic.com](https://console.anthropic.com))

### 1. Extract the model schema

In Power BI Desktop: **File → Save as → Power BI Project (.pbip)**. The model lands in
`<name>.SemanticModel/definition/` as TMDL. Flatten it into the text format shown in
[`schema.example.txt`](schema.example.txt) — tables, visible columns, measures with their DAX
expressions, relationships — and paste it into the `SCHEMAT` action.

This is the single highest-leverage step. See *Prompt quality* below.

### 2. Build the flow

Eleven actions, named exactly as in [`flow/expressions.md`](flow/expressions.md). Bodies for
both HTTP actions are in [`flow/`](flow/). **Test the flow fully in Power Automate before
opening Power Apps** — the run history shows the input and output of every action, which the
app does not.

### 3. Build the canvas app

Three controls are enough to see it working: a text input, a button, an HTML label. Positions,
sizes and formulas for the full layout are in the guide.

---

## Prompt quality is the whole game

The assistant is only as good as the schema it receives. Three things to do in Power BI Desktop
before blaming the model:

1. **Fill in Descriptions** on measures and dimension columns. This text goes straight into the
   prompt and acts as a business glossary. `[Margin %]` → *"gross margin percentage; use for
   profitability questions"*.
2. **Hide technical columns** — surrogate keys and sort columns only add noise.
3. **Strip auto-generated descriptions.** Tooling that writes *"Business attribute representing
   X within the Y entity"* for every column adds thousands of tokens and zero information. On a
   60-measure model, removing them cut the prompt from ~19 000 to ~3 100 tokens.

The system prompt in [`flow/system-prompt.txt`](flow/system-prompt.txt) also carries explicit
syntax rules for the mistakes that recur in practice — `FILTER` must wrap a table, `ORDER BY`
goes last, filtering on a measure result needs `VAR` + `FILTER`. Each of those was added after
watching the model get it wrong.

---

## Things that bite

Collected while building this. Each one cost real time.

- **`INFO.*` functions are not supported** by executeQueries, despite blog posts claiming
  otherwise — Microsoft excludes them in the Limitations section. The engine returns a generic
  `DatasetExecuteQueriesError`. That is why the schema comes from TMDL instead.
- **Measure expressions contain double quotes.** Interpolating the schema into a JSON request
  body without escaping produces `400 Bad Request` with no useful message. Hence the
  `replace()` chain in `PROMPT_JSON`.
- **Power Apps (V2) trigger inputs are not named what you called them.** The label is `pytanie`,
  the key is `text`. Symptom: the model politely explains it did not receive a question. Fix:
  a dedicated `PYTANIE` action, filled from *Dynamic content*, referenced everywhere after.
- **Claude returns a list of content blocks.** Indexing `content[0].text` breaks the moment the
  model emits a reasoning block first — use `last()`.
- **Gemini 3 models degrade below temperature 1.0.** Setting `temperature: 0` for determinism
  produced looping and `ServiceUnavailable` responses.
- **The Power BI connector reports bad DAX as `BadGateway`**, not as a syntax error. The real
  message is inside the action's output body.
- **A rectangle inserted after a label covers it.** Power Apps puts newly inserted controls on
  top regardless of insertion order — this looks exactly like a flow returning no data.

---

## Cost and limits

One question (DAX generation + answer) costs roughly **$0.007** with Claude Haiku — about
700 questions per $5. Most of it is the schema sent as input on every call.

| | |
|---|---|
| Power BI executeQueries | 100 000 rows / 1 000 000 values / 15 MB per query, 120 queries per minute |
| Power Automate (Developer Plan) | 750 flow runs per month, 100 actions per run |
| Power Apps response size | limited — hence the prompt forces small aggregated results |

---

## Security notes

- Queries are read-only by construction and validated before execution
- The API key lives in the HTTP header with **Secure Inputs** enabled; for production, move it
  to an environment variable of type *Secret* or Azure Key Vault
- Model metadata and **aggregated** result rows are sent to the LLM provider; raw fact rows
  never leave, because the prompt forbids returning them
- Where no data may leave the tenant, swap the two HTTP actions for Azure OpenAI or an internal
  endpoint — the rest of the flow is unchanged
- Don't commit your real schema: it contains every measure definition in your model

---

## Evaluation

[`docs/TESTY.md`](docs/TESTY.md) contains a 25-question test set graded from a single measure up
to decomposition questions, including **four questions that must fail** — asking about data the
model does not contain. A confident, well-written answer built on a wrong query is more
dangerous than a visible error, so those four matter as much as the rest.

Run the set after every prompt change, every model switch, and every change to the semantic model.

---

## Roadmap

- [ ] Repair loop as a Scope with *configure run after* (documented, not yet default)
- [ ] Few-shot examples of question → DAX pairs from the target model
- [ ] Query log — question, DAX, outcome — to find measures the model is missing
- [ ] Power Apps visual embedded in a Power BI report, consuming the report's filter context
- [ ] Azure OpenAI / internal-endpoint variant for tenants with strict data residency

---

## License

MIT — see [LICENSE](LICENSE).

---

<div align="center">

Built by **[@easy_analizy](https://instagram.com/easy_analizy)** · Power BI · Fabric · DAX

*If this saved you an afternoon, a star costs nothing.*

</div>
