# CDL — Cortex Detection Language

CDL is a small YAML language for writing web security checks. A rule says which HTTP requests to send and
what evidence would prove a finding. The rule engine sends the requests and decides, from the responses,
how strong that evidence is.

A rule is one `.yaml` file. It contains no code, and adding a rule does not require rebuilding the scanner.

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
steps:
  - id: "read"
    request: { method: "GET", path: "/demo/diagnostics" }
proof:
  kind: "signal.v1"
  assertions:
    all:
      - eq:       { left: { ref: "steps.read.status" }, right: { value: 200 } }
      - contains: { left: { ref: "steps.read.body" },   right: { value: "DEMO_DIAGNOSTICS_PAGE" } }
```

CDL is the rule format of **GGSec Cortex**. This repository holds the language documentation, the JSON
Schema, example rules, and the public rule catalogs. The engine that runs the rules is part of GGSec Cortex
and is distributed separately.

## What makes CDL different

**A rule cannot grade itself.** There is no "confidence" field. A rule picks a *proof profile* and the engine
decides the outcome. Matching text in a response is reported as *Likely*. *Confirmed* is only reached when the
engine took part in the test — for example by choosing random numbers, having the rule send them as an
expression, and finding the correct result in the response. See [proof profiles](docs/proof-profiles.md).

**"Could not tell" is a result.** A timeout, a truncated response or a skipped step never turns into "nothing
found". The rule ends *Inconclusive* and is listed as such, so an empty findings list is not mistaken for a
clean target.

**Rules are restricted on purpose.** A rule can only talk to the target it was given, within fixed request
and time budgets. The YAML itself is a small subset: no anchors, no tags, no duplicate keys, and an unknown
field is an error rather than something silently ignored.

**Rules are testable without a target.** Any rule can be run against a local file of canned responses, so
its behaviour can be checked in CI — including the case where it must stay quiet.

## Start here

| If you want to… | Read |
|---|---|
| write your first rule | [Authoring guide](docs/authoring.md) |
| look up a field, an operator or a limit | [Language reference](docs/language-reference.md) |
| understand *Likely* and *Confirmed* | [Proof profiles](docs/proof-profiles.md) |
| test a rule offline or in CI | [Testing rules offline](docs/testing.md) |
| ship rules to other people | [Packs and distribution](docs/packs.md) |
| get validation and completion in your editor | [`schema/cortex-rule-v1.schema.json`](schema/cortex-rule-v1.schema.json) |

## Repository layout

```
docs/               guides and the language reference
schema/             JSON Schema for a CDL rule
examples/           small, self-contained rules with fixtures - the best place to start reading
rules/official/     curated, product-specific rules, with offline test fixtures
rules/community/    rules ported from other open catalogs (signal-only; see its README)
```

`rules/official/` holds two kinds of rule. Some **demonstrate** a weakness — a file read or a command
execution the engine takes part in — and report *Confirmed*. Most **detect an affected version**: they match a
product's own version banner and report *Likely* through a `signal.v1` proof. Both kinds are product-specific
and ship with a positive and a negative fixture; a version match is evidence that a target is affected, not a
proof that it is exploitable.

## Try it

With the GGSec Cortex command line available as `ggsec`:

```bash
ggsec rules validate examples/
```

```bash
ggsec rules test examples/diagnostics-exposure.yaml --fixture examples/fixtures/diagnostics-exposure.json --expect Signal
```

```bash
ggsec rules test examples/challenge-arithmetic.yaml --fixture examples/fixtures/challenge-arithmetic.json --expect Confirmed
```

None of these commands sends a request anywhere. To use rules in a scan of a system you are authorised to
test:

```bash
ggsec --probe https://target.example --rule-engine yaml --rules-dir rules/official/
```

## Editor support

Point your editor's YAML language server at the schema to get completion and inline errors. For example, as
the first line of a rule file:

```yaml
# yaml-language-server: $schema=../schema/cortex-rule-v1.schema.json
```

The schema checks structure. `ggsec rules validate` additionally checks what a schema cannot: that
references resolve, that steps are in a valid order, and that the declared capabilities cover what the rule
uses.

## Status

The language is at `schema_version: 1` / `engine_api: 1`. A few behaviours described in the reference are
still being completed in the engine; they are listed in
[Conformance and current engine support](docs/language-reference.md#12-conformance-and-current-engine-support).

## Responsible use

These rules send real requests, and some of them change state or run commands on the target in order to prove
a finding. Run them only against systems you own or are explicitly authorised to test.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## Licence

The documentation, the schema, the examples and the rules in `rules/official/` are released under the
[Apache License 2.0](LICENSE); see also [NOTICE](NOTICE). Rules in `rules/community/` are derived from a
third-party catalog and keep the licence of their source; see
[`rules/community/README.md`](rules/community/README.md).
