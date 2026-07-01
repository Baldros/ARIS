# ARIS

**A deep-agent legal assistant for Brazilian law — that reasons like a lawyer, and never invents one.**

ARIS is being rebuilt from the ground up into an AI assistant that can take *any* question of Brazilian law and answer it the way a competent lawyer would: by going to the **primary sources**, reading the law that is actually in force, checking how the courts have applied it, and being honest about where doctrine is contested. It cites law number, article, and an official link — or it says it doesn't know.

> **Status: early rebuild.** Most of what this README describes is the *target*, not the current code. See [Project status](#project-status) for the honest breakdown of what exists today. This is a research/engineering project, not a finished product — and **nothing ARIS produces is legal advice.**

---

## The ambition

Most "legal AI" tools do one of two things: they either paste statute text into a chatbot and hope it stays current, or they build a giant frozen corpus of law and answer from it. Both go stale, both hallucinate, and both quietly break the moment they touch a field of law they weren't loaded with.

ARIS aims for something harder and more useful:

- **Work with *any* branch of Brazilian law** — not just the constitutional/criminal greatest-hits, but tax, energy regulation, labor, environmental, consumer, digital/data, agrarian, whatever the question demands.
- **Never store the law. Mine it live.** The statute you get is the statute in force *today*, pulled from the official source at question time — not a snapshot baked into a model months ago.
- **Reason across the classic tripod of legal work** — *lei seca* (the raw statute), *jurisprudência* (how courts apply it), and *doutrina* (scholarly interpretation) — treating each as a distinct source with its own reliability.
- **Refuse to hallucinate.** In law, a confident wrong citation is worse than "I don't know." Grounding and honest uncertainty are first-class design goals, not afterthoughts.

The bet: a legal assistant's value is not in *knowing* the law by heart, but in **knowing where the law lives and how to read it**. **Data mining is the core of this agent.**

---

## The core idea: mine, don't memorize

ARIS separates knowledge into two layers.

| | **Layer 1 — the Map** | **Layer 2 — the Law** |
|---|---|---|
| What | Taxonomy of Brazilian legal fields + where each field's law lives | The actual statutory / case / doctrinal text |
| Size | Small, curated, human-reviewable | Large, unbounded |
| Stored? | **Yes** — it's the compass | **No** — mined live, every time |
| Changes | Rarely (new field, new source) | Constantly (that's the point) |

**Layer 1 is a compass, not a memory.** For each *ramo* (branch) of Brazilian law it records the canonical *diplomas* (e.g. the CTN for tax, the CLT for labor), the official repositories where their current text lives (Planalto, LexML), the relevant courts (STF, STJ, TST…), the regulators that matter, and a set of routing keywords (`sinais`) that let ARIS classify a question into the right field *before* it spends a single web request.

That map already exists as a first draft: **[`knowledge/mapa_ramos_direito.yaml`](knowledge/mapa_ramos_direito.yaml)** — ~31 *ramos* grouped under the classic *summa divisio* (Público / Privado / Difuso-Social), each with its diplomas, official sources, tribunals, and `sinais`.

**Layer 2 is fetched on demand** from the sources Layer 1 points to. ARIS holds nothing between questions.

---

## Architecture: the tripod

The design is a **deep agent**: a coordinator that plans, delegates to focused specialists, and composes a grounded answer. The specialists mirror how legal work actually decomposes:

```
                 ┌──────────────────────────┐
   question ───▶ │   Coordinator (Aris)     │  classifies via the Map,
                 │   plans + composes        │  plans the search, grounds the answer
                 └────────────┬─────────────┘
                              │ delegates
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
    ┌───────────┐      ┌──────────────┐    ┌───────────┐
    │ Lei seca  │      │ Jurisprud.   │    │ Doutrina  │
    │ (statute) │      │ (case law)   │    │ (scholar) │
    └───────────┘      └──────────────┘    └───────────┘
   high confidence      high (official      low / heterogeneous
   Planalto/LexML       courts); caution     — no single official
                        with aggregators      source; always attributed
```

The three pillars are kept **separate on purpose** — they have different sources, different ranking logic, and very different failure modes. *Doutrina* in particular is the low-confidence pillar: there is no single official source for legal scholarship, so ARIS must attribute it and flag uncertainty rather than present it as settled law.

---

## Where it runs

ARIS is being built as an **Atlas agent**, hosted by `atlasd` through the ABI Shell. Consequences that shape the whole design:

- **Stateless per turn.** The host respawns the agent each turn and hands back the conversation history. There is no persistent server-side memory — which *fits* an agent that deliberately holds no law.
- **Model injected per turn.** The host supplies the LLM (`provider`, `model`, `key`) at request time. ARIS hardcodes **no** model and never freezes a provider key into the build — it is model-agnostic by construction.
- **Delegation-aware.** The platform exposes a supervisor/delegate channel and already mounts a live web-mining `Researcher` agent, which ARIS's mining layer may build on rather than reinvent.

---

## Project status

Honest accounting of the repository, right now:

**Exists and is the real starting point**
- ✅ `knowledge/mapa_ramos_direito.yaml` — the Layer 1 Map, **v0.1**. Drafted from working knowledge of Brazilian law; the diploma numbers and source URLs still need a verification pass against the live official sources before tooling is built on them.

**Legacy — kept for reference, *not* the target architecture (being replaced)**
- `app_Dejur.py` — a Streamlit prototype originally built as an internal legal-desk tool (DEJUR / Mitsui Gás). Discarded UI.
- `Agents.py`, `BehaviorSistem.py` — an Agno `Team` (coordinator "Aris" + `Oráculo`/`Ptolomeu`/`Hermes`) with a **hardcoded `gpt-4o`** and Streamlit-secret keys. This predates the atlasd reframe and violates the model-agnostic rule; it is scaffolding to learn from, not to keep.
- `WebTools.py` / `WebFunctions.py` — site-scraping logic. The **one salvageable piece**: this is the seed of the live-mining toolkit.
- `EmailTools.py`, `CLITools.py`, `chats.json`, `logo_mgeb_transparente.png` — legacy plumbing/branding, slated for removal.

**Not built yet (the roadmap below)**
- 🔜 The mining toolkit (Layer 2), the three specialists as designed, the atlasd conformance layer, and the anti-hallucination grounding checks.

---

## Design principles (non-negotiables)

1. **Don't store the law.** Mine the source of truth live, every time.
2. **Never hallucinate a citation.** Cite law number + article + official link, or declare uncertainty. Prefer "I don't know" to a confident guess.
3. **Official sources first.** Planalto/LexML for statute, official courts for case law. Aggregators (e.g. JusBrasil) are *secondary* and must be confirmed against the official source.
4. **Model-agnostic.** No hardcoded model, no frozen provider key — the host owns both.
5. **Any field, not a fixed set.** The Map's taxonomy is deliberately open-ended (a stable core plus a long tail of specialized/emergent fields).

---

## Roadmap

- [x] **Layer 1 — the Map** (v0.1 registry of Brazilian legal fields)
- [ ] Verify the Map's diplomas/URLs against live official sources
- [ ] Decide and lock the agent framework (constraint: **no LangChain/LangGraph**)
- [ ] Build the live-mining toolkit (Layer 2) — statute / case-law / doctrine fetchers
- [ ] Implement the tripod specialists + coordinator as a deep agent
- [ ] Conform to the Atlas agent standard (`agent.py`, `agentd_main.py`, `manifest.json`, `Skill/SKILL.md`)
- [ ] Anti-hallucination grounding & eval harness (citation checks, uncertainty flags)
- [ ] Retire the legacy Streamlit/Agno prototype

---

## Repository layout

```
ARIS/
├── knowledge/
│   └── mapa_ramos_direito.yaml   # Layer 1 — the Map (v0.1)  ← the real starting point
├── Agents.py                     # legacy Agno Team (to be replaced)
├── BehaviorSistem.py             # legacy agent behaviors/prompts
├── WebTools.py / WebFunctions.py # scraping logic (salvageable → mining toolkit)
├── app_Dejur.py                  # legacy Streamlit UI (discarded)
├── EmailTools.py / CLITools.py   # legacy tooling (to be removed)
└── requirements.txt
```

---

## Disclaimer

ARIS is an experimental engineering project. Its output is **not legal advice** and must not be relied upon as such. Always confirm against the official primary sources and consult a qualified lawyer for any real matter.
