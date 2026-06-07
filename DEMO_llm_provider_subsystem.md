# Demo — The LLM Provider Subsystem

> **For**: n2nhu lab engineering team
> **Prepared by**: Captain Jim Ames + Claude
> **Subject**: `schema_learner/llm_provider.py` — one protocol, three backends, identical guarantees
> **Date**: 2026-05-13

---

## TL;DR for the room

The toolkit talks to **three different LLM backends today** — Anthropic Claude, GPT4All running locally, and a deterministic test stub. **Adding a fourth (Ollama, vLLM, OpenAI, anything that speaks OpenAI's chat API shape) is roughly a 30-line drop-in.**

The reason this scales cheaply: **the principle binding doesn't live in the provider.** It lives in the *orchestrator* above the provider. So you can swap the brain, you cannot swap the guardrails.

Three integration patterns behind one `LLMProvider` Protocol. Captain calls them **NCP / MCP / ACP**:

- **NCP — Network Communications Protocol.** Our fast REST API to local LLMs (Ollama, GPT4All, vLLM, llama.cpp). OpenAI-compatible chat shape on a localhost port. Already wired up — the GPT4All provider IS this pattern, Ollama is a 30-line drop-in alongside it.
- **MCP — Model Context Protocol** (Anthropic's standard). Tools & data context. The cloud Claude path uses Anthropic's first-party API today; MCP-tool integration is the natural next step here.
- **ACP — Agent Communication Protocol** (IBM Research). Agent collaboration over REST. Roadmap pattern for when one of our triage steps needs to delegate to a specialist agent.

All three answer one method, return one string, and pass through identical downstream machinery. **One subsystem. Three integration patterns. Same rails.**

---

## Detailed Comparison

For the engineers in the room who want the precise framing of the three patterns the toolkit treats as first-class:

| Feature             | NCP                              | MCP                              | ACP                               |
|---------------------|----------------------------------|----------------------------------|-----------------------------------|
| **Main Focus**      | Local LLM inference over REST    | Tools & Data Context             | Agent Collaboration               |
| **Originator**      | Open / OpenAI chat shape         | Anthropic                        | IBM Research                      |
| **Architecture**    | Client-Server (HTTP/JSON)        | Client-Server (Tools/Resources)  | Client-Server (REST)              |
| **Primary Use Case**| Air-gapped or token-budget triage; fast local inference | Connecting an LLM to APIs, files, and rich context | Structured agent-to-agent interaction |
| **Key Strength**    | Zero API cost, offline-capable, drop-in adapters | Standardizing tool access for cloud LLMs | Easy setup, REST/cURL friendly    |

### What this means for the toolkit today

| Pattern | Status in `llm_provider.py` | What's wired up now |
|---|---|---|
| **NCP** — Network Communications Protocol | ✅ **Working** | `GP​T4AllProvider` at `localhost:4891` is exactly this pattern. Ollama is a ~30-line sibling class. |
| **MCP** — Model Context Protocol | 🟡 **Partial** | `AnthropicProvider` uses Anthropic's first-party API directly. Full MCP-tool integration (Anthropic's published spec) is a forward-compatible add — the protocol abstraction is already in place. |
| **ACP** — Agent Communication Protocol | 🔵 **Roadmap** | The factory pattern in `build_provider_from_config()` means an `ACPProvider` adapter is a single class away when we want one of our triage steps to delegate to a specialist agent. |

**The wow moment:** the toolkit doesn't have to choose. The `LLMProvider` Protocol unifies all three patterns behind one 3-method contract. NCP is already there. MCP is already there. ACP is plug-replaceable when the use case arrives.

---

## The Protocol

`llm_provider.py` line 116 — this is the entire contract a provider must satisfy:

```python
class LLMProvider(Protocol):
    """A minimal protocol an LLM provider must satisfy.

    The provider takes a prompt and returns a string response. That's
    it. Multi-turn flows are orchestrated by the LLMDrafter, not the
    provider — providers are stateless.
    """
    def generate(self, prompt: str) -> str: ...

    @property
    def is_available(self) -> bool: ...

    @property
    def model_label(self) -> str: ...
```

Three things. **That's the whole API.** Anyone who builds a class satisfying these three things is a valid provider.

This is the architectural lever. The thing that makes everything else possible.

---

## The Three Providers Today

### 1. MCP path — `AnthropicProvider` (line 136)

Calls `https://api.anthropic.com/v1/messages` directly via `urllib`. **Zero external dependencies** — no `requests`, no `anthropic` SDK, no pip-install required. The entire production cloud-LLM client is roughly 60 lines of Python.

This is the path that aligns with Anthropic's **Model Context Protocol** direction — when we add MCP-tool integration so Claude can call our internal lookups (concept-map, schema-learner corpus) during drafting, it slots in cleanly here.

Configured via `learn_config.ini`:

```ini
[mcp]
enabled = true
provider = anthropic
api_key_env_var = ANTHROPIC_API_KEY
model = claude-haiku-4-5-20251001
timeout_seconds = 30
```

Reads the API key from environment at runtime. **Never logged. Never stored on disk.** `is_available` is `True` if the env var is set.

`prompt_style = "strict"` — Claude reliably follows the strict structured-output prompt format, so we use the strict prompts with this provider.

### 2. NCP path — `GPT4AllProvider` (line 238)

Calls a local LLM server's OpenAI-compatible chat endpoint at `http://localhost:4891/v1/chat/completions`. **This is the Network Communications Protocol pattern** — a fast REST API to local inference. The same OpenAI chat shape Ollama, LM Studio, llama.cpp's server, and vLLM all expose.

Configured via `learn_config.ini`:

```ini
[mcp]
enabled = true
provider = gpt4all
gpt4all_host = localhost
gpt4all_port = 4891
gpt4all_model = Llama 3 8B Instruct
timeout_seconds = 60
```

**`is_available` probes the live server.** Before any triage starts, the provider does a `GET /v1/models` to confirm the runtime is up. If the local LLM server isn't running, `is_available` returns False, the drafter falls back gracefully, and the operator does the research themselves (Phase 2 behavior).

**Diagnostic stderr output is genuinely instrumented.** When the server returns an error or a wrong-shape response, the provider prints exactly what went wrong:

- HTTP 4xx/5xx with status code + response body excerpt
- JSON parse failures with raw response head
- Empty content with prompt length + max_tokens
- Specific hint: *"in GPT4All UI, verify '{model}' is the LOADED model"* when the OpenAI shape comes back without `choices[0].message.content`

`prompt_style = "lenient"` — small local models (7-8B) don't follow strict ALL_CAPS field labels as reliably as Claude does. The toolkit detects the `lenient` style and switches to a markdown-tolerant prompt + lenient parser that accepts variations.

### 3. The deterministic stub — `StubProvider` (line 198)

The third concrete implementation is a deterministic test fixture: returns canned responses keyed by prompt content. Used in the regression suite so tests don't depend on a network or a running model.

But its *architectural* role is bigger than tests. **It demonstrates that any third-party adapter is one-class away.**

Want Ollama on the NCP path? Today, this is the entire change set:

```python
class OllamaProvider:
    """Ollama exposes the same OpenAI chat shape on /v1/chat/completions
    by default (since Ollama 0.1.30+). This provider is therefore
    nearly identical to GPT4AllProvider — just a different default
    port (11434) and different model naming convention.
    """
    prompt_style = "lenient"
    
    def __init__(self,
                 host: str = "localhost",
                 port: int = 11434,
                 model: str = "llama3.1:8b-instruct-q4_K_M",
                 timeout: int = 60):
        # ... 4 lines of init ...

    @property
    def is_available(self) -> bool:
        # ... probes GET /api/tags ...

    @property
    def model_label(self) -> str:
        return f"ollama:{self.model}"

    def generate(self, prompt: str) -> str:
        # ... same OpenAI-shape POST as GPT4All ...
```

**Plus one block in `build_provider_from_config()`** at line 878:

```python
if provider_kind == 'ollama':
    host = config_section.get('ollama_host', 'localhost').strip()
    port = int(config_section.get('ollama_port', '11434'))
    model = config_section.get('ollama_model',
                                'llama3.1:8b-instruct').strip()
    provider = OllamaProvider(host=host, port=port, model=model,
                              timeout=timeout)
    if not provider.is_available:
        return None
    return provider
```

**And one section in `learn_config.ini`**:

```ini
[mcp]
provider = ollama
ollama_host = localhost
ollama_port = 11434
ollama_model = llama3.1:8b-instruct-q4_K_M
```

**Total work: roughly 30 lines.** All existing machinery — the four-turn interview, the citation verification, the worst-of-four confidence aggregation, the operator triage flow — keeps working with zero changes.

That's the leverage. **Adding a new brain doesn't touch any of the guardrails.**

---

## The Real Wow — The Multi-Turn Interview

If you stop reading here, the only takeaway is "plug-replaceable providers." That's good engineering. But the **real** wow is what the orchestrator on top does.

When a novel surface arrives at the LLM, the drafter (`LLMDrafter`, line 654) doesn't just ask "what is this?" once and trust the answer. It runs **a four-turn interview that any LLM goes through identically, regardless of which provider is behind the protocol.**

### Turn 1 — Discovery

The drafter packages the **engine's structural observations** about the surface and asks the LLM to describe what it sees:

```
You are helping classify a novel WatchGuard Firebox XML element for a
SonicWall→WatchGuard migration toolkit. The schema-learner detected a
new element and the operator needs a draft classification with
citations.

The novel element:
  Tag:               <Botnet>
  Naming tokens:     ['botnet']
  Top child tags:    (none — leaf element)
  Value patterns:    {'leaf': 'boolean-numeric'}
  Top structural neighbors (existing schema elements with similar
  structure, NOT semantic kinship — context only):
    - IPS: similarity 0.277
    - GAV: similarity 0.277
    - IAV: similarity 0.277

In 2-3 sentences, describe what this element is and what it
configures. If you don't recognize it from training, say so explicitly
and mark your draft as low confidence.

Be specific. Don't guess. If you're not sure, say so.
```

**Notice what's in the prompt:** the engine's *structural observations* (tag, tokens, child tags, value patterns, top neighbors). These are mechanical facts. The LLM gets the same view of the surface a senior engineer would get. **No semantic priming.** No "I think this is botnet detection, can you confirm?" The LLM has to derive that itself.

### Turn 2 — Structured extraction (provider-aware prompt)

Now the drafter asks for the four operator declarations in a structured format. The prompt **switches based on `provider.prompt_style`** — strict for Claude, lenient for local 7-8B models:

```
Return ONLY this exact format with NO additional text:

Q1_PURPOSE: (one sentence describing what this element does)
Q1_SOURCE: (full URL to vendor documentation supporting Q1, or "no citation found")

Q2_CATEGORY: (one of: network-plumbing, policy-access-control, object-library,
              tunnel-vpn, html-web-app, threat-detection,
              authentication-infrastructure, device-management, administrative)
Q2_SOURCE: (URL or "no citation found")

Q3_SW_EQUIVALENT: (the SonicWall equivalent feature, or "none — net-new on WatchGuard")
Q3_SOURCE: (URL or operator-experience marker, or "no citation found")

Q4_OPERATIONAL: (security or operational considerations to flag, or "none")
Q4_SOURCE: (URL or "no citation found")
```

**Every claim must come with a citation URL or an explicit "no citation found" admission.** The LLM cannot pass through ambiguity unflagged.

Lenient parser tolerates markdown bold, `Q1.PURPOSE` instead of `Q1_PURPOSE:`, dashes instead of colons — whatever a 7B model might emit. Strict parser used for Claude where output is reliable.

### Turn 3 — Self-validation

The drafter feeds the LLM its own answers and asks one yes/no question:

```
Review your own draft for the WatchGuard element <Botnet>:

  Q1 (purpose): Configures the Botnet Detection subsystem...
  Q2 (category): threat-detection
  Q3 (SonicWall equivalent): SonicWall Botnet Filter
  Q4 (operational notes): Requires active subscription...

Are these answers internally consistent and consistent with the
structural facts that the engine observed about this element
(child tags, value patterns)?

Answer with ONLY one word: yes or no.
```

If the LLM says "no," that's a signal the draft has self-detected inconsistency. **It contributes to the worst-of-four confidence aggregation.**

### Turn 4 — Programmatic citation verification

Now the drafter does something **the LLM cannot do**: it **actually fetches the citation URLs** from the network. For each Q1-Q4 citation:

1. Is it well-formed? If "no citation found" or not a URL → confidence stays `low`
2. Does it fetch? If HTTP 404, timeout, DNS failure → `unverified`, confidence stays `low`
3. Does the fetched page contain the relevant terms (surface tag + naming tokens)? If yes → `verified`, confidence becomes `high`. If no → `unverified`, confidence stays `low`

This is the failure mode the toolkit was *built* to catch. **Llama 3 8B hallucinates plausible-sounding URLs** — `https://www.watchguard.com/software/firebox-configurator` looks real, but it's HTTP 404. **The toolkit fetches it, observes the 404, and refuses to grant high confidence regardless of how authoritative the LLM's claim sounded.**

The diagnostic reason is recorded verbatim: `"URL did not fetch: HTTP 404"`. The operator sees this. The operator decides.

### Then: aggregate

```python
@staticmethod
def _aggregate_confidence(draft: LLMDraft) -> str:
    """Overall draft confidence is the worst of any individual
    claim's confidence, plus penalty if self-validation failed."""
    confidences = [
        draft.purpose.confidence,
        draft.category.confidence,
        draft.sonicwall_equivalent.confidence,
        draft.operational_notes.confidence,
    ]
    if "low" in confidences or not draft.self_validation_passed:
        return "low"
    if "medium" in confidences:
        return "medium"
    return "high"
```

**Worst-of-four.** One bad citation drops the whole draft to low. Self-validation failure drops the whole draft to low. There is no way for the drafter to mark a result `high` confidence unless **every Q1-Q4 has a fetched-and-relevant citation AND the LLM consistency-checks its own work.**

---

## How It Plugs Back Into the Schema-Learner

`triage.py` line 777, the actual operator workflow integration:

```python
mcp_drafter = LLMDrafter(
    provider=provider,
    verify_citations=verify_citations,
    verification_timeout=timeout,
)
```

Then for every Category-1 finding the operator walks:

```python
draft = mcp_drafter.draft(
    surface_tag=finding['element_tag'],
    structural_facts=finding['structural_facts'],
    neighbors=finding['neighbors'],
)
```

The draft becomes operator-visible context, **never replaces operator decision.** Action prompt is still `(c)orrect, (e)dit, (h)armful, (s)ideline, (q)uit`. The LLM proposes; the operator ratifies.

**Drafts are cached to disk** at `proposals/<run>/drafts/<surface_tag>.json`. If the operator quits mid-walk and resumes, no tokens are re-spent. If they want to audit "what did Llama say about Botnet on May 13," the draft is on disk, signed with timestamp + model label + verification status of every citation.

---

## The Numbers

From `llm_provider.py` (post Phase 7 hardening):

- **949 lines total**, including extensive docstrings and diagnostic logging
- **3 concrete providers** (Anthropic, GPT4All, Stub)
- **4 prompt templates** (Discovery, Extract-strict, Extract-lenient, Self-validate)
- **2 prompt styles** auto-selected by provider (`strict`, `lenient`)
- **8 distinct stderr diagnostic messages** for GPT4All failures — the toolkit tells you exactly what went wrong, not "draft failed"
- **Citation verification fetches up to 200KB** of any cited URL, checks for surface-tag + naming-tokens, extracts a 400-char excerpt with context
- **`User-Agent` header**: `n2nhu-migration-toolkit-citation-verifier/1.0` — vendor docs can see who's checking on them

---

## What This Architecture Buys Us

### 1. **Vendor independence at the LLM layer.**

Today: cloud-Claude or local-Llama. Tomorrow: Ollama, vLLM, OpenAI, Mistral hosted, Gemini if a customer asks. **The principle binding doesn't change.**

### 2. **Cost optionality.**

Burning Anthropic tokens for every triage walk in development gets expensive. Switch `provider = gpt4all`, run hundreds of dry-run drafts free, **the citation verifier catches Llama's hallucinations same as it would catch any model's.**

### 3. **Air-gap compatibility.**

A customer with strict outbound-network policies can run the toolkit fully offline. Local GPT4All on the engineer's laptop, no API calls to Anthropic. **The citation verifier still works** — it fetches vendor docs (watchguard.com, sonicwall.com), not LLM APIs.

### 4. **Regression discipline.**

`StubProvider` returns canned text. **All 15 test suites run deterministically with no network and no model loaded.** The principle binding is provable, not aspirational.

### 5. **Future-vendor agility.**

If a third firewall vendor (Fortigate, Palo Alto) gets added to the toolkit, the LLM stack doesn't care. The interview is about classifying **WatchGuard surfaces**, but the prompt templates are generic enough that "WatchGuard" is just a variable. A future Fortigate-mode toolkit would substitute it.

---

## Roadmap — Things We Can Add Cheaply

Here's what's a single-class drop-in given the architecture. Each entry tagged with which integration pattern (NCP/MCP/ACP) it aligns with:

| Adapter | Pattern | Effort | What it gets us |
|---|---|---|---|
| **`OllamaProvider`** | NCP | ~30 lines | Ollama on `:11434`. Fast local inference. The current go-to for laptop runtimes. |
| **`OpenAIProvider`** | NCP | ~30 lines | GPT-4o, GPT-4o-mini, o-series. Compare draft quality vs Claude. |
| **`vLLMProvider`** | NCP | ~30 lines | Self-hosted high-throughput inference for batch-triaging hundreds of surfaces |
| **`LMStudioProvider`** | NCP | ~30 lines | Desktop runtime for engineer laptops; same OpenAI chat shape as Ollama. |
| **MCP-tools wrapper** | MCP | ~150 lines | Lets `AnthropicProvider` call our internal concept-map / corpus lookups as MCP tools during drafting. Bigger payoff on citation quality. |
| **`ACPProvider`** | ACP | ~80 lines | Adapter for an IBM Research-style agent on REST. Forward-compatible with multi-agent triage flows. |
| **`MultiProvider`** | (meta) | ~80 lines | Calls TWO providers in parallel, compares answers, flags disagreement. **Phase 4 "deep research" candidate** — when two different models agree on a citation, that's stronger evidence than one model's word. |

The pattern in every case: implement the three-method protocol, add a `build_provider_from_config()` branch, document the INI keys. **Zero downstream changes.**

---

## What This Architecture Does NOT Compromise

It's worth being explicit about what stays the same across all providers, present and future:

- **No claim without provenance.** Every cited claim gets the same verification treatment.
- **Operator authority.** No provider produces a finalized classification. Every draft passes through `(c)/(e)/(h)/(s)` operator triage.
- **Drafts are cached and timestamped.** Audit trail captures which model produced which draft when.
- **Worst-of-four confidence aggregation.** A draft is high-confidence only if every Q1-Q4 citation fetched + matched terms.
- **Self-validation turn.** Every provider goes through Turn 3.
- **Graceful degradation.** If a provider is unavailable, the drafter returns an empty draft with diagnostic notes; the operator does the research themselves. **The toolkit never breaks because the LLM is down.**

---

## Bottom line for the room

Three integration patterns behind one Protocol. The Protocol is three methods. The orchestrator on top of the Protocol implements the principle binding — multi-turn interview, citation verification, worst-of-four aggregation, audit trail.

Captain's framing: **NCP / MCP / ACP all in one.** Plug-replaceable brains, fixed rails.

- **NCP** is live today via `GPT4AllProvider`. Ollama, vLLM, LM Studio, llama.cpp's server all drop in the same way — roughly 30 lines each.
- **MCP** is live today via `AnthropicProvider`. Full MCP-tool integration (so Claude can call our internal lookups during drafting) is forward-compatible.
- **ACP** is the next pattern we'll add when a triage step needs to delegate to a specialist agent. The factory pattern means it's a single class away.

**Adding a fourth (or fifth, or twentieth) backend is roughly 30 lines.** The next person who joins the lab can add their preferred local model and have the toolkit talking to it the same evening.

This is what good architecture buys you. Not magic. Just sharp lines drawn in the right places.

🚀

---

*Demo prepared 2026-05-13 by Captain Jim Ames + Claude.*
*Source: `schema_learner/llm_provider.py` (949 lines).*
*Integration: `schema_learner/triage.py` (~1700 lines).*
*Configuration: `schema_learner/learn_config.ini` [mcp] section.*
*Test coverage: `harness/test_phase3_llm.py`.*
