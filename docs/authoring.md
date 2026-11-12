# Authoring guide

This guide takes you from an empty file to a tested rule. It assumes you know HTTP and have read a YAML file
before; it does not assume you know CDL. The complete list of fields is in the
[language reference](language-reference.md).

- [1. The shape of a rule](#1-the-shape-of-a-rule)
- [2. Your first rule](#2-your-first-rule)
- [3. Check it without a target](#3-check-it-without-a-target)
- [4. Conditions](#4-conditions)
- [5. More than one step](#5-more-than-one-step)
- [6. Only run where it applies](#6-only-run-where-it-applies)
- [7. From "likely" to "confirmed"](#7-from-likely-to-confirmed)
- [8. Rules that change something](#8-rules-that-change-something)
- [9. Run it in a scan](#9-run-it-in-a-scan)
- [10. Checklist before you publish](#10-checklist-before-you-publish)
- [11. Common mistakes](#11-common-mistakes)

## 1. The shape of a rule

A rule answers four questions, always in the same places:

| Question | Where |
|---|---|
| What is this rule, and what does a finding mean? | `id`, `info`, `finding` |
| What is it allowed to do? | `requires`, `execution` |
| What does it send? | `steps` (and optionally `discovery`, `cleanup`) |
| What counts as proof? | `proof` |

One rule lives in one `.yaml` file.

## 2. Your first rule

This rule requests one page and reports a finding when the page looks like an exposed diagnostics screen. It
is [`examples/diagnostics-exposure.yaml`](../examples/diagnostics-exposure.yaml).

```yaml
schema_version: 1
id: "example:diagnostics-exposure"
rule_version: "1.0.0"
requires:
  engine_api: 1
  capabilities: ["http.standard.v1", "matchers.basic.v1", "proof.signal.v1"]
info:
  name: "Diagnostics endpoint exposed"
  description: "The application's diagnostics page is reachable without authentication."
  cwes: [200]
  tags: ["exposure", "example"]
  remediation: "Restrict the diagnostics endpoint to administrators or remove it from production builds."
finding:
  sink_type: "INFORMATION_DISCLOSURE"
  impact: "INFORMATION_DISCLOSURE"
  severity: "Low"
  root_cause: "diagnostics-exposure"
execution:
  auth: "anonymous"
  side_effects: "read_only"
  transport: "http"
  max_requests: 1
steps:
  - id: "read"
    request:
      method: "GET"
      path: "/demo/diagnostics"
proof:
  kind: "signal.v1"
  assertions:
    all:
      - eq:
          left: { ref: "steps.read.status" }
          right: { value: 200 }
      - contains:
          left: { ref: "steps.read.body" }
          right: { value: "DEMO_DIAGNOSTICS_PAGE" }
```

Reading it top to bottom:

- **`id`** is `namespace:name`. Use a namespace you own — your organisation's name is a good choice. The
  `cortex.*` namespaces belong to the catalogs in this repository.
- **`requires.capabilities`** lists the engine features the rule uses: plain HTTP, conditions, and the
  `signal.v1` proof. If you use a feature without listing it, validation tells you which one is missing.
- **`info`** is what a person reads in the report. Write `remediation` for the person who has to fix it.
- **`finding`** says what a hit means. `root_cause` is a short stable name for the underlying problem; do not
  change it between versions.
- **`execution`** declares that the rule is anonymous and only reads.
- **`steps`** has one step, called `read`, that sends one request. Paths are relative to the target; a rule
  cannot name another host.
- **`proof`** says: status 200 *and* the body contains the marker.

Notice what is missing: the rule does not say how confident it is. `signal.v1` means "this is an observation",
and the engine reports it as *Likely*. You will see in [section 7](#7-from-likely-to-confirmed) how a rule
earns *Confirmed*.

## 3. Check it without a target

```bash
ggsec rules validate examples/diagnostics-exposure.yaml
```

Validation catches typos, unknown fields, references to steps that do not exist, and missing capabilities,
each with a line and column.

```bash
ggsec rules inspect examples/diagnostics-exposure.yaml
```

`inspect` prints what the rule would send, without sending it.

To see the rule decide, give it canned responses — a *fixture*:

```json
{
  "responses": [
    { "method": "GET", "path": "/demo/diagnostics", "status": 200, "body": "<h1>DEMO_DIAGNOSTICS_PAGE</h1> uptime 4d 2h" }
  ],
  "default": { "status": 404, "body": "not found" }
}
```

```bash
ggsec rules test examples/diagnostics-exposure.yaml --fixture examples/fixtures/diagnostics-exposure.json --expect Signal
```

Then test that it stays quiet when the page is not there:

```bash
ggsec rules test examples/diagnostics-exposure.yaml --fixture examples/fixtures/diagnostics-exposure.absent.json --expect NoMatch
```

[Testing rules offline](testing.md) describes fixtures in full.

## 4. Conditions

Conditions appear in `proof.assertions`, in a step's `when` and `expect`, and in `preconditions`. They are
always written the same way: an operator, a `left`, and a `right`.

```yaml
contains:
  left:  { ref: "steps.read.body" }      # a value observed during the run
  right: { value: "Server uptime" }      # a literal you write
```

`ref` points at something the run produced; `value` is a constant. Both sides can be either.

| Operator | Use |
|---|---|
| `eq`, `ne` | equal, not equal |
| `gt`, `gte`, `lt`, `lte` | compare integers |
| `in` | the left value is one of a list: `right: { value: [200, 201] }` |
| `contains` | substring; add `ignore_case: true` if needed |
| `regex` | regular expression match |
| `exists` | `exists: { ref: "steps.read.headers.x-powered-by" }` |
| `all`, `any`, `not` | combine other conditions |

The most used references:

```
steps.<id>.status          steps.<id>.body          steps.<id>.headers.<name>
extracts.<name>            variables.<name>
```

**Conditions have three answers, not two.** If the data a condition needs is missing — the request timed
out, the step was skipped — the answer is *unknown*, and the rule reports `Inconclusive` rather than
pretending nothing is there. You do not have to write anything for this; it is how every condition behaves.

## 5. More than one step

Steps run in order, and a later step can use what an earlier one returned. Two tools make that possible:
**extractors** pull a value out of a response, and **templates** (`{{ … }}`) put a value into a request.

```yaml
steps:
  - id: "create"
    request:
      method: "POST"
      path: "/demo/notes"
      json_body:
        text: "cdl-example {{ variables.marker }}"
    expect:
      in:
        left: { ref: "steps.create.status" }
        right: { value: [200, 201] }
    extract:
      note_id:
        json_pointer:
          from: "steps.create.body"
          pointer: "/id"
          required: true
  - id: "read"
    request:
      method: "GET"
      path: "/demo/notes/{{ extracts.note_id }}"
```

- **`expect`** states what a healthy response to *this* step looks like. It is not proof of anything; it only
  says "carry on". If it fails, the rule stops trusting this step and ends `Inconclusive`.
- **`extract`** names a value (`note_id`) and says where to find it. Besides `json_pointer` there are `regex`
  and `header` extractors.
- **`{{ extracts.note_id }}`** inserts that value into the next request. Templates work in `path`, in header
  values, in `body`, and in the strings of a `json_body`.

Templates can call a few functions, for example `{{ url_encode(variables.name) }}` or
`{{ concat('id-', extracts.note_id) }}`. The full list is in the
[reference](language-reference.md#54-templates).

A request is only sent when everything its template needs is known. If `note_id` could not be extracted, the
`read` request is not sent at all — the engine never fills a gap with an empty string.

**Variables** give a rule values of its own:

```yaml
variables:
  marker:  { generator: "nonce" }        # a fresh random token for every run
  section: { value: "reports" }          # a constant the operator may override
```

Use a generated `nonce` or `uuid` whenever the rule needs to recognise its own data.

## 6. Only run where it applies

A rule written for one product should not send its requests to every site. `discovery` runs a cheap request
first, and `preconditions` decides from the answer whether the rest of the rule runs.

```yaml
discovery:
  - id: "home"
    request: { method: "GET", path: "/" }
preconditions:
  contains:
    left: { ref: "steps.home.headers.server" }
    right: { value: "ExampleServer" }
    ignore_case: true
```

When the gate does not hold, the rule ends `NotApplicable` and none of its main steps are sent.

Inside the main steps, `when` does the same job for a single step: the step is skipped unless the condition
holds.

## 7. From "likely" to "confirmed"

A `signal.v1` rule can match text. It cannot tell a vulnerable server from one that merely prints the same
words. To report a finding as *Confirmed*, a rule uses a **confirming proof profile**, in which the engine
takes part in the test.

[`examples/challenge-arithmetic.yaml`](../examples/challenge-arithmetic.yaml) shows the idea:

```yaml
steps:
  - id: "probe"
    request:
      method: "POST"
      path: "/demo/calc"
      json_body:
        expr: "{{ challenge.a }}*{{ challenge.b }}"
  - id: "control"
    request:
      method: "POST"
      path: "/demo/echo"
      json_body:
        expr: "{{ challenge.a }}*{{ challenge.b }}"
proof:
  kind: "expression_evaluation.v1"
  challenge: { kind: "integer_product" }
  readback:  { step: "probe", ref: "steps.probe.body" }
  control:   { step: "control" }
```

The engine picks two random numbers for this run. The rule sends them as `a*b` but never knows the product.
The engine then looks for the product in the response. A server that only repeats its input returns `a*b`,
not the product, so it cannot pass; the `control` step shows the product does not simply appear everywhere.

Try both outcomes:

```bash
ggsec rules test examples/challenge-arithmetic.yaml --fixture examples/fixtures/challenge-arithmetic.json --expect Confirmed
```

```bash
ggsec rules test examples/challenge-arithmetic.yaml --fixture examples/fixtures/challenge-arithmetic.echo.json --expect NoMatch
```

There are four confirming profiles — for file reads, expression evaluation, command execution, and
out-of-band callbacks. [Proof profiles](proof-profiles.md) explains each, what it claims, and how to choose.
The rules in [`rules/official/`](../rules/official/) are worked examples of the file-read and command-execution
profiles; [`examples/challenge-arithmetic.yaml`](../examples/challenge-arithmetic.yaml) is one for expression
evaluation; the out-of-band profile is described, with its shape, in [proof profiles](proof-profiles.md).

## 8. Rules that change something

If a rule creates, modifies or deletes anything on the target, say so and clean up:

```yaml
execution:
  auth: "anonymous"
  side_effects: "state_change"
  transport: "http"
cleanup:
  - id: "remove"
    request:
      method: "DELETE"
      path: "/demo/notes/{{ extracts.note_id }}"
    expect:
      in:
        left: { ref: "steps.remove.status" }
        right: { value: [200, 202, 204] }
```

`cleanup` steps run after the main steps whether or not the rule found anything, and even if it was
interrupted. The `expect` on a cleanup step tells the engine whether the cleanup really worked; the result is
shown in the report next to the finding.

Three habits keep such rules safe:

- **Make what you create recognisable.** Put a generated `nonce` or `uuid` in it.
- **Delete by the id you created or extracted**, never by a name someone else's data might share.
- **Do the least that proves the point.** A rule needs evidence, not impact.

The complete example is [`examples/resource-lifecycle.yaml`](../examples/resource-lifecycle.yaml).

## 9. Run it in a scan

```bash
ggsec --probe https://target.example --rule-engine yaml --rules-dir ./my-rules
```

Only scan systems you are authorised to test. Every rule the scan considered is listed in its coverage
report with the status it ended in, including the ones that did not apply or could not decide — an empty
findings list is never the whole story.

## 10. Checklist before you publish

- [ ] `ggsec rules validate` reports no errors and no warnings.
- [ ] A fixture where the weakness is present gives the expected outcome.
- [ ] A fixture where the target is patched, or only echoes, gives `NoMatch`.
- [ ] The rule was run against a real vulnerable instance in a lab, and against a fixed one.
- [ ] `proof.kind` is no stronger than the evidence: text matches are `signal.v1`.
- [ ] `side_effects` is truthful, and anything created is removed in `cleanup`.
- [ ] `preconditions` keep a product-specific rule away from unrelated targets.
- [ ] `info.description` says what was observed; `remediation` says what to do.
- [ ] `id` is in your own namespace, and `root_cause` is a name you will keep.

## 11. Common mistakes

| Symptom | Cause |
|---|---|
| `unknown field '…'` | A typo, or a field in the wrong place. Unknown fields are errors by design. |
| `expected a mapping` on `left` or `right` | A condition written with a bare string. Write `{ ref: "…" }` or `{ value: "…" }`. |
| `forward reference to step '…'` | A `when` or a request template refers to a step that has not run yet. |
| `rule uses an operation needing capability '…'` | Add that capability to `requires.capabilities`. |
| The rule validates but always ends `Inconclusive` | A step's `expect` is failing, or a template needs a value that was never extracted. The transcript from `rules test` shows which. |
| A number comparison never matches | The literal is quoted. `{ value: 200 }` is a number; `{ value: "200" }` is text. |
| A JSON field arrives as a string instead of a number | A templated `json_body` value is always a string. |
| A `signal.v1` rule matches pages it should not | The assertion is too loose. Require the status *and* a marker specific to the product. |
