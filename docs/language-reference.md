# CDL language reference

This is the complete reference for a CDL rule at `schema_version: 1`, `engine_api: 1`. It defines every
field, type, expression and limit. For a gentler start, read the [authoring guide](authoring.md) first; for
how findings are graded, read [proof profiles](proof-profiles.md).

A rule is one UTF-8 `.yaml` (or `.yml`) file, at most 256 KiB, holding exactly one YAML document.

## Contents

1. [The YAML a rule may use](#1-the-yaml-a-rule-may-use)
2. [Top-level fields](#2-top-level-fields)
3. [The result model](#3-the-result-model)
4. [Steps and requests](#4-steps-and-requests)
5. [Expressions: conditions, references, templates](#5-expressions-conditions-references-templates)
6. [Variables and generators](#6-variables-and-generators)
7. [Transport, scope and limits](#7-transport-scope-and-limits)
8. [Proof](#8-proof)
9. [Cleanup](#9-cleanup)
10. [Capabilities](#10-capabilities)
11. [Identity](#11-identity)
12. [Conformance and current engine support](#12-conformance-and-current-engine-support)

---

## 1. The YAML a rule may use

CDL uses a deliberately small subset of YAML, so a rule is predictable and cannot carry surprises. The loader
reads the parser event stream and rejects anything outside the subset **before** any value is interpreted.

**Allowed:** mappings with plain string keys, sequences, and scalars (`string`, `int`, `bool`, `null`).

**Rejected, as a hard error with line and column:** anchors (`&`), aliases (`*`), merge keys (`<<`), explicit
tags (`!!str`, `!foo`), more than one document (a second `---`), non-string or complex keys, and **duplicate
keys at any depth**. Nesting may not exceed 24 levels, and a rule may not contain more than 20 000 nodes.

Scalar types are never guessed. In a `string` field the text is kept exactly as written (so `on`, `yes`, `1.0`
stay strings). A `bool` field accepts only `true` or `false`. Numbers parse in the invariant culture. An
**unknown field or an out-of-range enum value is an error**, never ignored — a typo fails the rule instead of
silently doing nothing.

How a literal is typed depends on how you write it:

| You write | Type |
|---|---|
| `"200"`, `'true'` (quoted) | string |
| `200`, `-5` (unquoted integer) | int |
| `true`, `false` (unquoted) | bool |
| `null`, `~`, or nothing | null |
| any other unquoted text | string |

> **Quote strings that look like something else.** Write `rule_version: "1.0.0"` and
> `{ value: "200" }` when you mean text; write `{ value: 200 }` when you mean a number.

Both block style and flow style (`{ method: "GET", path: "/" }`) are accepted and mean the same thing.

---

## 2. Top-level fields

| Field | Required | Meaning |
|---|---|---|
| `schema_version` | yes | Must be `1`. |
| `id` | yes | `"<namespace>:<local-id>"`; each part matches `^[a-z0-9][a-z0-9._-]*$`. The namespace marks who owns the rule; see [§11](#11-identity). |
| `rule_version` | yes | Quoted SemVer, e.g. `"1.2.0"`. |
| `requires` | yes | `{ engine_api: 1, capabilities: [ ... ] }`. Every capability the rule uses must be listed; see [§10](#10-capabilities). |
| `info` | yes | Human-readable metadata; see below. |
| `finding` | yes | What a positive result means; see below. |
| `execution` | yes | How the rule is allowed to run; see [§7](#7-transport-scope-and-limits). |
| `variables` | no | Named constants and generators; see [§6](#6-variables-and-generators). |
| `preconditions` | no | A cheap gate that decides whether the rule applies at all; see [§4](#4-steps-and-requests). |
| `discovery` | no | Bounded recon steps run before `preconditions`. |
| `steps` | yes | The ordered HTTP steps, at least one; see [§4](#4-steps-and-requests). |
| `proof` | yes | How a finding is proven; see [§8](#8-proof). |
| `cleanup` | no | Compensation steps; see [§9](#9-cleanup). |

Map key order carries no meaning anywhere; sequence order is the only ordering.

**`info`** holds `name`, `description`, `remediation` (all required) and optional `references` (URLs),
`cves` (strings), `cwes` (integers) and `tags` (strings). Everything in `info` is rendered as text; CVEs and
CWEs are metadata, not identity.

**`finding`** holds:

- `sink_type` — a free string naming the weakness category, e.g. `CODE_EXEC`, `PATH_TRAVERSAL`,
  `INFORMATION_DISCLOSURE`.
- `impact` — one of: `RCE`, `FILE_READ`, `PRIV_ESC`, `AUTH_BYPASS`, `SESSION_THEFT`,
  `INFORMATION_DISCLOSURE`, `EMAIL_ABUSE`, `DOS`, `WEAK_TOKEN`, `MITM_RISK`, `FINANCIAL_INTEGRITY`,
  `CREDENTIAL_COMPROMISE`, `NONE`.
- `severity` — one of `Info`, `Low`, `Medium`, `High`, `Critical`.
- `cvss_vector` — optional CVSS vector string.
- `root_cause` — a short, stable slug for the underlying cause; part of the finding's identity ([§11](#11-identity)).

There is **no `certainty` field**. A rule never states how sure it is — the engine decides that from the proof
([§3](#3-the-result-model)). Declaring `certainty` is an error.

---

## 3. The result model

Each execution produces three independent values. A rule sets none of them.

- **ExecutionStatus** — did the rule run? `Completed`, `NotApplicable`, `SkippedPolicy`,
  `UnsupportedCapability`, `InvalidRule`, `Inconclusive`, `Error`, `Cancelled`.
- **DetectionOutcome** — what was seen? `NoMatch`, `Signal`, `Confirmed`, `Undetermined`.
- **CleanupStatus** — was anything left behind? `NotRequired`, `NotAttempted`, `Succeeded`, `Failed`,
  `Unknown`.

| ExecutionStatus | Meaning |
|---|---|
| `Completed` | The rule ran to the end and the proof was decided. |
| `NotApplicable` | `preconditions` did not hold; no main request was sent. |
| `SkippedPolicy` | The rule was not allowed to run here (for example its auth context is not available). |
| `UnsupportedCapability` | The rule needs a capability this engine does not provide. |
| `InvalidRule` | The rule failed validation and was never executed. |
| `Inconclusive` | The rule ran but could not decide: a timeout, a truncated response, an exhausted budget, a step whose `expect` failed. |
| `Error` | An unexpected failure inside the engine. |
| `Cancelled` | The scan was stopped while the rule was running. |

| DetectionOutcome | Meaning |
|---|---|
| `NoMatch` | The proof was decided and did not hold. |
| `Signal` | An observation consistent with the weakness. |
| `Confirmed` | The weakness was proven by a confirming proof profile. |
| `Undetermined` | The proof could not be decided either way. |

`CleanupStatus` is described in [§9](#9-cleanup).

Only `Signal` and `Confirmed` become findings:

| DetectionOutcome | Reported certainty |
|---|---|
| `Signal` | **Likely** — an observation consistent with the weakness. |
| `Confirmed` | **Confirmed** — proven by a confirming proof profile. |

Every other combination is coverage information, not a finding. In particular, `Completed + NoMatch` means
only "this rule did not match here", never "the target is safe". A positive finding whose `CleanupStatus` is
`Failed` keeps its evidence and additionally reports the residual state — one axis never hides another.

---

## 4. Steps and requests

A rule runs in a fixed order:

```
discovery steps  →  preconditions gate  →  steps  →  proof  →  cleanup steps
```

A **step** is `{ id, when?, request | variants, expect?, extract?, repeat? }`. The `id` is unique across the
whole rule (discovery, steps and cleanup together) and matches `^[a-z0-9][a-z0-9_-]*$`. There is no goto and
no arbitrary loop; steps run in sequence order.

- **`when`** (optional [condition](#51-conditions)) is evaluated **before** the step, over data from earlier
  steps only. Referencing this step or a later one is a validation error. If `when` is false or cannot be
  decided, the step is skipped and everything it would have produced is unknown.
- **`expect`** (optional condition) is the success check for **this** step after its response arrives. It is a
  liveness/applicability check, never proof of a weakness. When `expect` does not hold, the step counts as
  not completed: its observations are unknown to later conditions and to the proof, so the rule ends
  `Inconclusive` rather than `NoMatch`.
- **`extract`** pulls values out of the response for later steps; see [§5.3](#53-extractors).
- **`repeat`** polls a read; see [§6](#6-variables-and-generators).

**`request`** fields:

| Field | Meaning |
|---|---|
| `method` | `GET`, `HEAD`, `POST`, `PUT`, `PATCH`, `DELETE`, `OPTIONS`. |
| `path` | Begins with `/`, relative to the target origin. No `//host`, no scheme, no authority. |
| `headers` | Map of string → string. Values may use [templates](#54-templates). Names and values are rejected if they contain CR, LF or NUL. |
| `body` | Exact request body text (templated). Mutually exclusive with `json_body`. An absent body is not an empty body. |
| `json_body` | A typed map/list serialized to JSON **after** template substitution, so a templated value becomes a proper JSON string. Sending `json_body` sets `Content-Type: application/json` unless the rule set it. |
| `redirects` | `"none"` (default), or `{ same_origin: true, max: N }` with `N` from 1 to 4. A redirect to another origin is never followed. |
| `path_mode` | `"standard"` (default) or `"preserve_raw"`. `preserve_raw` sends `%xx`, dot-segments and the query exactly as written (needed for some path-normalisation bugs) and requires the `http.preserve_raw_path.v1` capability. |

**`variants`** — a step may carry `variants: [ {request...}, ... ]` instead of one `request`: a finite list
of requests, each with the same fields as `request`. The variants are tried in order; the first one that
completes and satisfies the step's `expect` is accepted and the rest are skipped. All variants share the step
id, so later references see the accepted response. Give such a step an `expect` — without one, the first
variant that gets any response is accepted. This models one weakness reachable by several equivalent paths
without an open-ended loop. Every variant counts against the request budget.

**`discovery`** is a list of recon steps with the same shape, run before `preconditions`. Its requests count
against the request budget. Discovery being read-only does not by itself permit later state-changing steps —
those still need `execution.side_effects`.

**`preconditions`** is a single [condition](#51-conditions) over discovery results (a banner, a fingerprint).
It decides whether the rule applies. When the gate does not hold, the whole rule is `NotApplicable` and
nothing further is sent — neither the main steps nor cleanup — so a product-specific rule never probes an
unrelated target. This includes a gate that cannot be decided because the fact it asks about is simply
absent, such as a header the target does not send. If a discovery request got no answer at all (a timeout, a
network error), the engine does not know whether the rule applies, and the rule ends `Inconclusive` instead.

---

## 5. Expressions: conditions, references, templates

### 5.1 Conditions

A condition is a typed tree used by `when`, `expect`, `preconditions`, `repeat.until` and
`proof.assertions`. Nesting is at most 16 deep.

Combinators: `all: [ ... ]`, `any: [ ... ]`, `not: <condition>`.

Leaves take `left` and (where shown) `right`, each written as `{ ref: "<path>" }` **or** `{ value: <literal> }`
— never a bare string, and never anything executable:

| Leaf | Meaning |
|---|---|
| `eq`, `ne` | Equal / not equal, type-exact (string vs string, int vs int). |
| `gt`, `gte`, `lt`, `lte` | Integer comparison. |
| `in` | `left` is one of `right`, where `right` is `{ value: [ ... ] }`. |
| `contains` | `left` (string) contains `right` (string). Add `ignore_case: true` for case-insensitive. |
| `regex` | `left` (string) matches the pattern in `right`. Patterns run with a 100 ms timeout; a regex error is never a match. |
| `exists` | The single `ref` resolves to a present value. |

Missing data is never an empty string. An unresolved `ref` is **unknown**, so the leaf is unknown and the
branch is `Undetermined`, not false. A negative check (such as `not contains`) over a **truncated** response
body is `Undetermined`, never a pass. In `all`/`any`, the validator still checks every sub-node even when
runtime short-circuits.

### 5.2 References

A `ref` names a value by path:

```
steps.<step-id>.status            int     HTTP status code
steps.<step-id>.headers.<name>    string  header value (case-insensitive name; repeats joined)
steps.<step-id>.body              string  response body (decompressed, size-capped)
steps.<step-id>.elapsed_ms        int     round-trip time
steps.<step-id>.transport_state   string  success | timeout | network_error | blocked
steps.<step-id>.truncated         bool    whether the body hit the size cap
extracts.<name>                   string  a value extracted by an earlier step
variables.<name>                  the variable's value
challenge.a, challenge.b          int     operands of an expression_evaluation.v1 proof (see §8)
challenge.nonce                   string  token of a command_output_readback.v1 proof (see §8)
oob.url, oob.host, oob.token      string  the engine's out-of-band callback address (see §8)
target.origin                     string  e.g. "https://host:8443"
target.base_path                  string  the app's base path, "" when none
```

A reference can only look backwards: `when` and request templates see earlier steps; `expect`, `extract` and
`repeat.until` also see the current step; `preconditions` sees discovery steps only; `proof` sees every step.
A reference to a step that was skipped or did not complete is unknown.

### 5.3 Extractors

A step's `extract` maps `name → extractor`; each name is unique across the whole rule. An extracted value is
immutable once set and at most 64 KiB.

- `regex`: `{ regex: { from: "<ref>", pattern: "...", group: "<group>", required?: bool, cardinality?: "exactly_one" | "first" } }`
  — the value is the named (or numbered) capture group of the match.
- `json_pointer`: `{ json_pointer: { from: "<ref>", pointer: "/a/b/0", required?: bool } }` — RFC 6901
  pointers only. A JSON string yields its text; any other JSON value yields its raw JSON.
- `header`: `{ header: { from: "steps.<id>.headers.<name>", name: "<name>", required?: bool } }` — the
  value of that response header.

Extractors run only when their step completed. A value that could not be extracted is unknown: a later
request whose template needs it is not sent, and conditions over it cannot be decided, so the rule ends
`Inconclusive`. `required: true` additionally records the missing value in the transcript.

### 5.4 Templates

Templates appear in `request.path`, header values, `request.body`, and `json_body` string leaves, written
`{{ … }}`. A template is substituted after the step's `when` passes and before the request is sent. There is
no `eval`, no reflection and no file or environment access.

An expression is a reference, a string literal `'...'`, an integer, or a call to one of these functions:

```
url_encode(x)  json_escape(x)  base64(x)  lower(x)  upper(x)
concat(a, b, ...)  add(a, b)  sub(a, b)  mul(a, b)
```

Arithmetic is checked 64-bit; overflow is an error. Whitespace inside `{{ }}` is trimmed. A literal `{{` is
written `{{ '{{' }}`. An unterminated `{{` is a validation error, never passed through silently.

Three things worth knowing:

- **A template never renders a blank.** If an expression refers to a value that is unknown at that moment
  (a skipped step, a failed extract), the request is not sent at all.
- **In `json_body`, a templated value is always a JSON string.** `count: 5` sends the number `5`;
  `count: "{{ variables.n }}"` sends a string. Rely on serialization rather than escaping by hand —
  `json_escape` is only for building JSON inside a plain `body`.
- **Templates are not expanded in conditions or patterns.** Use `{ ref: ... }` there instead.

---

## 6. Variables and generators

`variables` maps a name to `{ value: <literal> }` or `{ generator: <kind> }`:

- `nonce` — cryptographically random, URL-safe. `{ generator: "nonce", bytes?: 8..32 }` (default 16).
- `uuid` — a random UUID (version 4).
- `random_int` — `{ generator: "random_int", min: <int>, max: <int> }`, cryptographic and bounded.

Generators never use a fixed seed in a scan. Variables are immutable and isolated per (target, rule, scan): no
variable or extracted value ever leaks across rules, targets or scans. A `value` variable can be overridden by
the operator at scan time; a generator cannot (that would defeat the proof that depends on it).

### `repeat` — polling a read

`repeat: { max_attempts: 1..5, delay_ms: 0..60000, until?: <condition> }` re-runs a step up to `max_attempts`,
waiting `delay_ms` between attempts, and stops as soon as `until` holds (or, with no `until`, as soon as the
read completes). Every attempt counts against the budget and the overall deadline still applies. `repeat` is
for reads only; a state-changing request is never retried automatically after an unknown outcome.

---

## 7. Transport, scope and limits

Every request returns a typed `transport_state`: `success`, `timeout`, `network_error` or `blocked` (stopped
by policy). A transport error is never treated as a clean empty response. A response body larger than the
size cap is cut off and flagged `truncated`.

**Scope.** A rule's requests are limited to the **target origin**, even if the surrounding scan covers more
hosts. The origin is re-checked after every redirect; a response can never redirect the rule to another host.
Private/lab addresses are valid scope and are not blocked as if they were SSRF.

**Auth** (`execution.auth`): `anonymous`, `current`, `guest`, `admin`.

- `anonymous` carries no cookies, no `Authorization`, and never inherits the scan's session.
- `current`, `guest`, `admin` require a matching profile to be configured; there is no silent fallback (for
  example `guest` never quietly becomes `admin`). A missing profile makes the rule `SkippedPolicy`.
- The result records the auth context actually used.

**Side effects** (`execution.side_effects`) declare the most intrusive thing the rule does:

| Value | The rule… |
|---|---|
| `read_only` | only reads. |
| `state_change` | creates, changes or deletes something on the target, or sends a crafted expression. |
| `remote_command` | makes the target run an operating-system command. |
| `oob` | makes the target call out to the engine's out-of-band collaborator. |

This is a declaration for the operator and for review — be honest in it; `GET` does not by itself mean "no
side effects". The validator ties it to the proof: `command_output_readback.v1` requires `remote_command`,
`expression_evaluation.v1` requires `state_change` or `remote_command`, and `oob_interaction.v1` requires
`oob`.

**Transport** (`execution.transport`) is always `"http"`.

**Request headers.** The engine sets `Host`, `Content-Length` and the connection framing itself. A rule must
not set them, and must not carry credentials in headers — authentication comes only from `execution.auth`.

**Limits a rule runs under** (a rule may make them smaller via `execution.max_requests` /
`execution.timeout_seconds`, never larger):

| Limit | Value |
|---|---|
| Rule file size | 256 KiB |
| YAML depth / nodes | 24 / 20 000 |
| Steps (discovery + main + cleanup) | 64 |
| Requests (discovery + main) | 32 |
| Cleanup requests | 8 |
| Per-request time | 10 s |
| Main + discovery total time | 60 s |
| Cleanup total time | 15 s |
| Response body (after decompression) | 1 MiB |
| Regex match | 100 ms timeout |

Retries and discovery count against the budget. A negative check over a truncated response is `Undetermined`.
A parser, regex or transport error is never a positive match.

---

## 8. Proof

A rule names exactly one `proof.kind`. The rule declares the observations a profile needs; the **engine**
decides `Confirmed`. A signature or trusted source never raises certainty. The five profiles are specified in
full, with fixtures, in [proof profiles](proof-profiles.md); in brief:

- **`signal.v1`** — an assertion tree over responses. A match is a `Signal` (Likely). Never `Confirmed`.
- **`file_read_differential.v1`** — a clean baseline without a file-structure marker, and a probe that has it,
  proves a file read.
- **`expression_evaluation.v1`** — an integer the engine chose and never sent literally comes back → confirmed
  server-side evaluation.
- **`command_output_readback.v1`** — an engine nonce plus characteristic command output, absent from a control
  and from what was sent → confirmed command execution.
- **`oob_interaction.v1`** — an out-of-band callback on an engine-owned token within a time window → confirmed
  interaction.

The challenge values a proof provides (`challenge.a`, `challenge.b`, `challenge.nonce`) and the OOB bindings
(`oob.url`, `oob.host`, `oob.token`) are referenced from templates but never set by the rule.

---

## 9. Cleanup

`cleanup` is an ordered list of steps that compensate for what a state-changing rule did. It is **not** a
transactional rollback. Once a main step has been attempted, cleanup runs whatever happened next — success,
no match, timeout, error or cancellation — under its own time budget (15 s) and its own request reserve (8),
in the same scope and auth context. If no main step was attempted (the rule was `NotApplicable`, for
example), cleanup does not run. A cleanup step has one `request` and an optional `expect` that checks the
compensation actually took effect. A finding is finalized after cleanup, so the report can show the residual
state when cleanup did not succeed.

| CleanupStatus | Meaning |
|---|---|
| `NotRequired` | The rule changed nothing and declares no cleanup. |
| `Succeeded` | Every cleanup step completed and its `expect` held. |
| `Failed` | A cleanup step completed but its `expect` did not hold — something is probably left behind. |
| `Unknown` | The engine cannot tell: a cleanup request failed in transport or could not be built, or the rule changed state but declares no cleanup. |
| `NotAttempted` | Cleanup was needed but could not be started. |

Write cleanup so that it can only touch what the rule itself made: create something identifiable (put a
generated `nonce` or `uuid` in it), address it by the id the rule generated or extracted, and delete exactly
that. Never delete by a guessable or shared name. A cleanup request whose template needs a value the run
never produced is not sent. The [resource-lifecycle example](../examples/resource-lifecycle.yaml) shows the
shape.

---

## 10. Capabilities

`requires.capabilities` lists the mechanisms a rule uses. A capability gates which operations are allowed; it
does not run code. Every operation a rule uses must be covered by a listed capability, and every listed
capability must be available.

| Capability | Enables |
|---|---|
| `http.standard.v1` | `method`/`path`/`headers`/`body`/`json_body`, `redirects`, extractors, `repeat` |
| `matchers.basic.v1` | the condition operators in [§5.1](#51-conditions) |
| `http.preserve_raw_path.v1` | `path_mode: preserve_raw` |
| `proof.signal.v1` | `proof.kind: signal.v1` |
| `proof.file_read_differential.v1` | `proof.kind: file_read_differential.v1` |
| `proof.expression_evaluation.v1` | `proof.kind: expression_evaluation.v1` |
| `proof.command_output_readback.v1` | `proof.kind: command_output_readback.v1` |
| `proof.oob_interaction.v1`, with `oob.dns.v1` and/or `oob.http.v1` | `proof.kind: oob_interaction.v1` |

Listing a capability the rule does not use is harmless; using an operation without listing its capability,
or listing a name the engine does not know, is a validation error. Two names are reserved for mechanisms that
are not available yet (`http.read_malformed.v1`, `oob.http_raw.v1`); a rule that lists them does not validate.

---

## 11. Identity

**Rule identity** is the `id`. The part before the colon is the namespace and says who owns the rule:

| Namespace | Used for |
|---|---|
| `cortex.official` | The curated catalog in [`rules/official/`](../rules/official/). Reserved. |
| `cortex.community` | Rules ported from other open catalogs, in [`rules/community/`](../rules/community/). Reserved. |
| anything else | Your own rules. Pick something you control, such as your organisation's name (`acme`, `acme.web`). |

An id must be unique in the catalog a scan loads. `rule_version` changes when the rule changes; the id does
not.

**Finding identity** is derived from the target, the rule's namespace and `root_cause`, and the sink and
impact. It does **not** include the rule version or any random value of a run (a generated id, a challenge),
so the same weakness found by two versions of a rule is the same finding. Keep `root_cause` stable once a
rule is published. Because identity includes the namespace, two publishers that pick the same `root_cause`
keep separate findings.

When a rule **confirms** a weakness that carries a CVE in `info.cves`, signal-only findings from other rules
for the same CVE on that target are folded into the confirmed one, so one issue is reported once.

Reports carry the rule id, version and proof kind alongside the finding, so a result is traceable to the exact
rule that produced it.

---

## 12. Conformance and current engine support

The reference above describes the language contract. A rule written to it stays valid as the engine fills in
the last behaviours. At the time of writing, these points are worth knowing — they are being completed, and
none changes the YAML you write:

- **Auth.** The current engine executes rules in the `anonymous` context only. Support for the `current`,
  `guest` and `admin` profiles is being added; until it lands, publish only rules that are meaningful without
  a session, and still declare `auth` truthfully so the rule needs no change later.
- **Extractor `cardinality`.** Accepted, but the current engine always takes the first match. Declare the
  value you mean.
- **Selecting rules by side effect.** `side_effects` is recorded and checked against the proof today;
  letting the operator allow or deny whole classes of side effect per scan is being added. Until then, choose
  what runs by choosing which rules you load.
- **OOB proof** requires a configured out-of-band collaborator. With none, the proof is `Undetermined` and no
  request is sent — the engine never fabricates a callback.

The authoritative check for any rule is `ggsec rules validate` (structure and semantics) followed by a
`ggsec rules test` fixture run ([testing](testing.md)).

## Versioning

`schema_version` identifies the rule format and `engine_api` the set of behaviours a rule may rely on. Both
are `1`. Additions that keep existing rules valid (a new capability, a new proof profile, a new template
function) do not change them; a change that would alter the meaning of an existing rule gets a new number, and
engines keep running version 1 rules as specified here.
