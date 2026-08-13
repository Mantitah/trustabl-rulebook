---
policy_id: claude_sdk_tool_definition
category: claude_sdk
topic: tool_definition
rules:
  - id: CSDK-001
    severity: low
    confidence: 0.95
    scope: tool
    fix_type: code
  - id: CSDK-002
    severity: medium
    confidence: 0.9
    scope: tool
    fix_type: code
  - id: CSDK-007
    severity: low
    confidence: 0.9
    scope: tool
    fix_type: code
  - id: CSDK-008
    severity: medium
    confidence: 0.8
    scope: tool
    fix_type: code
  - id: CSDK-014
    severity: low
    confidence: 0.9
    scope: tool
    fix_type: code
  - id: CSDK-017
    severity: low
    confidence: 0.85
    scope: tool
    fix_type: code
  - id: CSDK-018
    severity: low
    confidence: 0.8
    scope: tool
    fix_type: code
references: [LLM06]
---

# Policy Rationale: Tool Definition Hygiene

**Policy ID:** `claude_sdk_tool_definition`  
**File:** `claude_sdk/tool_definition.yaml`  
**Rules:** CSDK-001, CSDK-002, CSDK-007, CSDK-008, CSDK-014, CSDK-017, CSDK-018  
**Severities:** low, medium, low, medium, low, low, low  
**Fix types:** code, code, code, code, code, code, code  
**References:** LLM06

---

## What this policy covers

The surfaces the model reads to decide *whether and how* to call a Claude Agent
SDK tool: its description, its parameter types, and its name. These rules fire on
`@tool` / `@claude_tool` / `claude_agent_sdk`-decorated functions (and MCP tool
registrations, for the description and name checks). CSDK-001 fires when the
function has no docstring (predicate `has_docstring: false`); CSDK-002 when no
parameter is type-annotated (`has_typed_params: false` with params present);
CSDK-007 when the function name is a vague verb like `process`, `handle`, `run`,
`execute`, or `do` (`name_in`). CSDK-017 and CSDK-018 check description
*quality* rather than description *presence* — a tool can pass CSDK-001 (it has
a docstring) and still give the model nothing usable, either because the text
is a placeholder stub (CSDK-017) or because it is too short to carry a
selection signal (CSDK-018).

---

## Why tool-definition hygiene is a distinct concern in agent tools

In a conventional library, a function's name and types are a convenience for the
human who calls it. In an agent, they are the *entire* interface the model uses
to route a request: Claude reads the tool's name, its docstring (which the SDK
surfaces verbatim as the tool description), and its parameter schema, then
decides which tool to invoke and what arguments to synthesize. There is no human
in that loop to disambiguate.

When the description is missing, the model must guess the tool's purpose from the
name alone — and under an ambiguous prompt it mis-selects, calling the wrong tool
or calling the right one with wrong arguments. When parameters are untyped, the
model has no schema to constrain the values it fabricates, so it passes a string
where an int was meant, or invents a shape the function then mishandles. When the
name is a generic verb, two tools named `process` and `run` are
indistinguishable to the router. Each gap degrades tool selection precisely when
the input is unclear — which is exactly when correct routing matters most.

These are reliability and correctness issues rather than direct exploits, but in
an agentic setting a mis-selected tool *is* an unintended action — which is why
the cluster anchors loosely to OWASP LLM06 (Excessive Agency): poor tool
boundaries let the model act in ways the author did not intend.

CSDK-017 and CSDK-018 are, as of this writing, the only rule pair in this
policy with direct empirical backing rather than first-principles reasoning
about the SDK's routing mechanism: Hasan et al., *MCP Tool Descriptions Are
Smelly!* (arXiv 2602.14878), audited 856 real-world MCP tool descriptions
across 103 servers and found 97.1% carried at least one description-quality
defect, with 56% failing to state the tool's purpose clearly. That finding is
the empirical case for treating description *quality* as a distinct,
independently-worth-checking failure mode from description *presence* —
CSDK-001 alone would have passed nearly all of the smelly descriptions the
study found, because a placeholder or a five-word stub still satisfies
`has_docstring: true`.

---

## Rule-by-rule defense

### CSDK-001 — Tool has no description (Severity: low, Confidence: 0.95, Fix type: code)

**What we detect:**
A decorated Claude SDK tool (or MCP tool) whose function has no docstring
(`has_docstring: false`).

**Why it is flaggable:**
The SDK uses the docstring as the description shown to the model. With none, the
model routes on the function name alone.

**Real-world consequence:**
A tool `def lookup(q: str)` with no docstring sits next to `def search(q: str)`;
the model cannot tell which retrieves what, and picks wrong under an ambiguous
query — returning the wrong data with full confidence.

**Why severity is low and not medium:**
It degrades selection quality but rarely causes direct harm on its own, and a
well-named tool partially compensates. It is high *confidence* (0.95) because the
absence of a docstring is unambiguous, but low *impact*.

**Fix type — code:**
Add a docstring to the function — a source edit.

**Confidence 0.95:**
The only false positive is a tool whose description is supplied through a
decorator argument rather than the docstring (uncommon in this SDK); otherwise a
missing docstring is exactly what it looks like.

### CSDK-002 — Tool parameters are not type-annotated (Severity: medium, Confidence: 0.9, Fix type: code)

**What we detect:**
A tool with at least one parameter where no parameter carries a type annotation
(`has_typed_params: false`, params present). `self`/`cls` are ignored.

**Why it is flaggable:**
Type annotations become the JSON schema the model fills. Without them the model
has no constraint on argument shape and fabricates loosely-typed values.

**Real-world consequence:**
`def transfer(amount, to)` with no types lets the model pass `amount="a lot"` or
a malformed account id; the tool then coerces or crashes mid-side-effect.

**Why severity is medium and not low:**
Untyped arguments cause *wrong-argument* execution, not just mis-selection — the
tool runs with bad data. It is not high because the failure usually surfaces as
an error rather than a silent breach.

**Fix type — code:**
Annotate the parameters — a source edit.

**Confidence 0.9:**
False positives: a tool that derives its schema from a Pydantic model passed
elsewhere may be safe yet annotation-free in the signature. Uncommon enough to
hold at 0.9.

### CSDK-007 — Ambiguous tool name (Severity: low, Confidence: 0.9, Fix type: code)

**What we detect:**
A tool whose name is one of a closed set of vague verbs (`process`, `handle`,
`run`, `execute`, `do`, …) via `name_in`.

**Why it is flaggable:**
A generic name carries no routing signal; the model cannot distinguish it from
any other generic-named tool.

**Real-world consequence:**
Two tools named `run` and `process` in the same agent are a coin-flip for the
router whenever the prompt does not name one explicitly.

**Why severity is low and not medium:**
A clear docstring can rescue a vague name, so impact is bounded; this is a
clarity nudge, not a defect that breaks execution.

**Fix type — code:**
Rename to a specific verb-noun (`fetch_invoice`, `restart_worker`) — a source
edit.

**Confidence 0.9:**
The name list is curated, so matches are deliberate; the small false-positive
space is a domain where `run` is genuinely descriptive (rare).

---

### CSDK-008 — Tool exposes **kwargs without explicit input_schema (Severity: medium, Confidence: 0.8, Fix type: code)

**What we detect:** a tool whose accepted arguments live under `**kwargs` (a
parameter named `kwargs`) with no `input_schema=` on the `@tool` decorator
(`param_name_matches exact:[kwargs]` AND `not tool_decorator_kwarg_present:[input_schema]`).

**Why it is flaggable:** the SDK derives the model-facing JSON schema from the
signature; a `**kwargs`-only tool exposes an empty parameter object, so the model
gets no signal about which keys to send.

**Real-world consequence:** the model omits required keys or invents unhandled
ones; the failure surfaces as a runtime `KeyError` at invoke time instead of a
clean schema-validation error before the tool runs.

**Why severity is medium and not low:** unlike a missing docstring this produces
wrong-argument *execution*, not just mis-selection. Not high because it usually
fails loudly rather than silently breaching anything.

**Fix type — code:** declare each parameter on the signature with a type, or pass
an explicit `input_schema=` (JSON Schema dict or Pydantic model).

**Confidence 0.8:** a tool that genuinely uses a documented `input_schema` yet
still names a `kwargs` param could fire. Discovery surfaces the `**kwargs` splat
name (as well as a plain param named `kwargs`), so the rule fires on a real
`**kwargs` signature.

---

### CSDK-014 — TypeScript Claude SDK tool has no description (Severity: low, Confidence: 0.9, Fix type: code)

**What we detect:**
A TypeScript Claude Agent SDK `tool(name, description, schema, handler)` whose
`description` (the second positional argument) is empty (`has_docstring: false`).
Discovery captures the description only when that argument is a plain string
literal: `tsStringLiteralText` returns the unquoted text for a tree-sitter
`string` node and an empty string for anything else, and `PredHasDocstring` is
`TrimSpace(Description) != ""`. So both an omitted/empty literal **and** a
description built from a non-literal expression (a template string with `${...}`,
an identifier, a member access, or a concatenation) are captured as empty and
fire. Unlike the Python sibling CSDK-001, which reads the description from the
function docstring, the TypeScript factory takes it as an explicit argument.

**Why it is flaggable:**
The SDK passes this `description` to the model as the sole signal for when to
invoke the tool. With it empty, the model routes on the tool name alone — the
exact mis-selection mechanism documented for the Python sibling
[CSDK-001](#csdk-001--tool-has-no-description-severity-low-confidence-095-fix-type-code).

**Real-world consequence:**
A TypeScript `tool("lookup", "", schema, handler)` sits next to a
`tool("search", "...", ...)`; under an ambiguous query the model cannot tell which
retrieves what and picks wrong, returning the wrong data with full confidence.

**Why severity is low and not medium:**
Like CSDK-001 it degrades selection quality but rarely causes direct harm on its
own, and a well-chosen tool name partially compensates. Matches the Python
sibling's low severity.

**Fix type — code:**
Supplying the `description` argument is an edit to the `tool(...)` call site —
tool source.

**Confidence 0.9:**
Marginally below the Python sibling's 0.95. The detection is mechanically exact (a
literal description is captured, anything else reads as empty), so the firing
itself is unambiguous; the 0.9 reflects that a non-literal description built at
runtime (a constant assembled from `const` fragments) is genuinely present to the
model yet captured as empty here — a false positive the literal-only capture
cannot rule out. It carries no `re.compile`-style collision, so it does not drop
lower.

---

### CSDK-017 — Tool description is a placeholder (Severity: low, Confidence: 0.85, Fix type: code)

**What we detect:**
A tool whose description contains a placeholder marker — `todo`, `tbd`,
`fixme`, `placeholder`, `no description`, or `does stuff` — matched
case-insensitively against the full description text
(`has_description_text: [todo, tbd, fixme, placeholder, "no description",
"does stuff"]`). Unlike CSDK-001, this rule does not require the docstring to
be absent; it fires on a docstring that exists but is a stub.

**Why it is flaggable:**
The SDK forwards the description to the model verbatim as its selection
signal. A stub like `"""TODO: describe this tool."""` passes CSDK-001's
`has_docstring: true` check yet gives the model exactly as little to route on
as no description at all — the marker text describes the author's unfinished
work, not the tool.

**Real-world consequence:**
A tool `def lookup(q: str)` docstringed `"""TODO"""` sits next to `def
search(q: str)` with a real description; the model has no way to know
`lookup` even exists as an option and either ignores it or calls it on a
guess, same as the CSDK-001 failure mode but hidden from a docstring-presence
check.

**Why severity is low and not medium:**
Same reasoning as CSDK-001: it degrades selection quality without directly
causing harm, and a distinctive function name partially compensates.
Confidence is *higher* than CSDK-001 despite the lower severity — see below.

**Fix type — code:**
Replace the placeholder with a real description — a source edit.

**Confidence 0.85:**
The marker list is curated and deliberately narrow (`todo`, `tbd`, `fixme`,
`placeholder`, `no description`, `does stuff`), so a match is rarely
accidental — a legitimate description that happens to contain the substring
`"todo"` (e.g. `"Sync items to the user's todo list"`) is the main false-positive
class, which is why this sits a notch below CSDK-001's 0.95 rather than above
it despite the narrower trigger set.

### CSDK-018 — Tool description is too short to guide model selection (Severity: low, Confidence: 0.8, Fix type: code)

**What we detect:**
A tool that has a docstring (`has_docstring: true`, guarding against
double-firing alongside CSDK-001) whose trimmed description is under 40
characters (`description_length_lt: 40`). Both conditions must hold
(`all:`).

**Why it is flaggable:**
Forty characters is roughly the length of `"Gets data."` or `"Processes the
request."` — long enough to exist, far too short to say what the tool does,
what it returns, or when to prefer it over a similarly-named neighbor. The
model gets a description-shaped string with none of the information a
description exists to carry.

**Real-world consequence:**
`def get_user(id: str) -> dict:` docstringed `"""Gets a user."""` (16 chars)
sits next to `get_user_preferences` and `get_user_permissions`; the model
cannot tell from either description which one to call for a "what can this
user do" query and either picks wrong or calls all three.

**Why severity is low and not medium:**
As with CSDK-001 and CSDK-017, this degrades routing precision rather than
causing a direct failure, and a specific function name can partly compensate
for a thin description.

**Fix type — code:**
Expand the description — a source edit.

**Confidence 0.8:**
The lowest confidence in this policy, and the threshold is the reason: 40
characters is a length heuristic, not a semantic judgment of sufficiency. A
tool with self-explanatory parameter names (`def convert_usd_to_eur(amount:
float) -> float:` docstringed `"""Converts USD to EUR."""`, 21 chars) may
need little prose because the signature itself carries the routing signal —
a real false-positive class this rule cannot distinguish from a genuine stub.
The guard against double-firing with CSDK-001 (`has_docstring: true`) is exact,
not probabilistic; the 0.8 reflects the length threshold alone.

---

## What this policy does not cover

- Descriptions or names that are present but *misleading* — a docstring that
  describes the wrong behavior passes CSDK-001 but mis-routes worse than none.
- Parameter types that are present but too loose (`x: Any`, `data: dict`) — they
  satisfy CSDK-002 yet give the model little schema to work with.
- Overlapping tool *purposes* (two distinct, well-named tools that nonetheless do
  near-identical things) — a design issue no single-tool predicate sees.
- Descriptions supplied via decorator kwargs rather than the docstring.
- For CSDK-014: a TypeScript description assembled from a non-literal expression
  (a `const` reference, a template string, a concatenation) is real text the model
  sees, but the literal-only capture records it as empty and fires anyway.
- Whether a description is *accurate* — CSDK-017 and CSDK-018 check that a
  description is present and long enough, not that it correctly describes
  what the tool does. A confidently-worded, 200-character description of the
  wrong behavior passes both.
- The 40-character floor in CSDK-018 is a length heuristic, not a semantic
  one; it does not account for tools whose parameter names already carry most
  of the routing signal and can be genuinely self-explanatory at a length
  the rule still flags.

---

## Recommendations beyond the fix

```python
from claude_agent_sdk import tool

@tool
def fetch_invoice(invoice_id: str, include_lines: bool = False) -> dict:
    """Fetch a single invoice by its ID.

    Use this when the user asks about one specific invoice and gives its ID.
    Returns the invoice header; set include_lines=True to also return line
    items. Does not search — use search_invoices for lookups by date or amount.
    """
    ...
```

1. Write the docstring for the *router*, not the human: say when to use the tool
   and, crucially, when **not** to (point at the sibling tool that handles the
   other case).
2. Type every parameter, and prefer narrow types (`Literal[...]`, an enum, a
   constrained `int`) over `str`/`Any` so the model's argument space is small.
3. Name tools `verb_noun` and keep the verbs distinct across the tool set;
   reserve generic verbs for nothing.
