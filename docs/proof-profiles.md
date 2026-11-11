# Proof profiles

A CDL rule does not say how sure it is. It says **what must be observed**, by choosing one proof profile in
`proof.kind`, and the engine decides the outcome from the responses it actually received.

That split is the point of the language. A scanner that lets every check grade itself ends up with reports
where "confirmed" means whatever the check's author hoped. In CDL the grade comes from a small, fixed set of
profiles, each with a stated claim and a stated way to fail.

| Profile | Strongest outcome | What a positive result claims |
|---|---|---|
| [`signal.v1`](#signalv1) | Signal (reported as *Likely*) | The response looks like the weakness. |
| [`file_read_differential.v1`](#file_read_differentialv1) | Confirmed | The target returned file contents it should not. |
| [`expression_evaluation.v1`](#expression_evaluationv1) | Confirmed | The server evaluated an expression we supplied. |
| [`command_output_readback.v1`](#command_output_readbackv1) | Confirmed | The server ran an operating-system command we supplied. |
| [`oob_interaction.v1`](#oob_interactionv1) | Confirmed | The target contacted an address only this run knew. |

Rules that hold for every profile:

- **Unknown is not "no".** If a step the proof needs timed out, was skipped, or failed its `expect`, the
  outcome is `Undetermined` and the rule ends `Inconclusive` — never a clean `NoMatch`.
- **Absence needs a complete response.** "The marker is not there" is only decided on a response that was
  not truncated.
- **A regex error or timeout is never a match.**
- **Where a rule comes from does not matter.** A signed, official rule gets exactly the same grading as a
  local one.

Each profile needs its capability listed in `requires.capabilities`, shown under the profile's name.

---

## `signal.v1`

Capability: `proof.signal.v1`

```yaml
proof:
  kind: "signal.v1"
  assertions:
    all:
      - eq:       { left: { ref: "steps.read.status" }, right: { value: 200 } }
      - contains: { left: { ref: "steps.read.body" },   right: { value: "Server uptime" } }
```

`assertions` is any [condition](language-reference.md#51-conditions) over the rule's steps.

| Assertions | Outcome |
|---|---|
| true | `Signal` |
| false | `NoMatch` |
| cannot be decided | `Undetermined` |

A signal is an observation: a banner, an exposed page, a version string, an error message. It can never be
`Confirmed`, however specific the assertion — matching text shows what the response looks like, not what the
server did. Use it for exposure and misconfiguration findings, and for any check where a differential or a
challenge is not possible.

Example: [`examples/diagnostics-exposure.yaml`](../examples/diagnostics-exposure.yaml).

---

## `file_read_differential.v1`

Capability: `proof.file_read_differential.v1`

```yaml
proof:
  kind: "file_read_differential.v1"
  baseline: { step: "baseline" }
  probe:    { step: "probe" }            # or a list: [ { step: "probe_a" }, { step: "probe_b" } ]
  marker:   { regex: "<pattern that only the file's contents match>" }
```

- `baseline` — a step that asks the same endpoint for something harmless.
- `probe` — the step (or steps) that asks for the file. With a list, the vectors are tried in the order
  given and the first whose response carries the marker confirms; several vectors stay one finding.
- `marker` — a regular expression describing the *structure* of the file, not a word that could appear on any
  page.

The engine confirms when the baseline completed, the marker is **absent** from the baseline, and the marker is
**present** in a probe response.

| Situation | Outcome |
|---|---|
| Marker absent in baseline, present in a probe | `Confirmed` |
| Marker already present in the baseline | `NoMatch` — the page shows it anyway, nothing was read |
| Marker absent in every probe | `NoMatch` |
| Baseline did not complete, or no probe completed, or a probe was truncated before the marker | `Undetermined` |

The claim is a **file read**. It is not, by itself, code execution; set `impact` to `FILE_READ` or
`INFORMATION_DISCLOSURE` (the validator warns otherwise).

---

## `expression_evaluation.v1`

Capability: `proof.expression_evaluation.v1` · requires `side_effects: state_change` or `remote_command`

```yaml
proof:
  kind: "expression_evaluation.v1"
  challenge: { kind: "integer_product" }
  readback:  { step: "probe", ref: "steps.probe.body" }
  control:   { step: "control" }
```

For every run the engine picks two random integers and makes them available as `{{ challenge.a }}` and
`{{ challenge.b }}`. The rule places them in an expression in the target's own syntax — for example
`{{ challenge.a }}*{{ challenge.b }}`. The engine computes the product itself and never gives it to the rule.

- `readback` — the step, and the reference inside it, where the result should appear.
- `control` — a step whose response must **not** contain the product. Typically the same input sent somewhere
  it is only stored or echoed, or a request without the expression.

The engine confirms when the product appears in the readback, does **not** appear in the control, and was
**not** present in what the rule sent.

| Situation | Outcome |
|---|---|
| Product in readback, not in control, not in the sent request | `Confirmed` |
| Product not in the readback | `NoMatch` |
| Product also in the control | `NoMatch` — the value is there regardless |
| Product was in the sent request | `NoMatch` — an echo, not an evaluation |
| Readback or control did not complete, or readback truncated before the value | `Undetermined` |

Send the operands, never the result: a rule that computes the product itself (for instance with `mul`) and
sends it proves nothing and is rejected in review.

The claim is **server-side expression evaluation**. That is often the first step towards code execution, but
this profile does not show an operating-system command ran; use `command_output_readback.v1` for that.

Example: [`examples/challenge-arithmetic.yaml`](../examples/challenge-arithmetic.yaml), with a fixture that
evaluates and a fixture that only echoes.

---

## `command_output_readback.v1`

Capability: `proof.command_output_readback.v1` · requires `side_effects: remote_command`

```yaml
proof:
  kind: "command_output_readback.v1"
  challenge: { kind: "nonce" }
  readback:
    step: "read_output"
    ref: "steps.read_output.body"
    marker: { regex: "<pattern of the command's own output>" }
  control: { step: "control" }
```

For every run the engine issues a random token, `{{ challenge.nonce }}`. The rule makes the injected command
print that token and also run something whose output has a recognisable shape (the identity of the current
user, for example).

- `readback.marker` — a regular expression for the *command's output*, something the request itself does not
  contain.
- `control` — a step whose response must not match the marker.

The engine confirms when the readback contains the token **and** matches the marker, the control does **not**
match the marker, and the matched output was **not** present in what the rule sent.

| Situation | Outcome |
|---|---|
| Token and marker in readback; marker absent from control and from the sent request | `Confirmed` |
| Token missing from the readback | `NoMatch` — the output is unrelated or stale |
| Token present but marker missing | `NoMatch` — the request was reflected, nothing ran |
| Marker also in the control | `NoMatch` |
| Readback or control did not complete, or readback truncated | `Undetermined` |

The step that carries the token and the step that reads the output may be different steps: a rule can start a
job with one request and collect its output with a later one, using
[`repeat`](language-reference.md#6-variables-and-generators) to wait for it.

The claim is **remote command execution**, on the platforms the rule's command works on. A rule should say in
`info.description` which platforms those are.

---

## `oob_interaction.v1`

Capabilities: `proof.oob_interaction.v1` and `oob.http.v1` or `oob.dns.v1` · requires `side_effects: oob`

```yaml
proof:
  kind: "oob_interaction.v1"
  protocol: "http"          # or "dns"
  window_seconds: 10        # 1..30, default 15
```

For every run the engine creates a callback address that belongs to this rule, this target and this scan, and
exposes it as `{{ oob.url }}` (for HTTP) and `{{ oob.host }}` (for DNS). The rule places it where the target
might fetch or resolve it. After the steps finish, the engine waits up to `window_seconds` for a callback.

| Situation | Outcome |
|---|---|
| A callback on this run's address, on the declared protocol, inside the window | `Confirmed` |
| No callback | `NoMatch` |
| No out-of-band collaborator is configured for the scan | `Undetermined` — and no request is sent |

A callback on the wrong protocol does not count: a DNS lookup does not prove an HTTP fetch. The engine's own
health-check traffic is excluded.

The claim is **the interaction**: the target reached out to an address it could only have learned from this
rule. What that means — a server-side request, a blind injection — is the rule's stated `sink_type` and
`impact`, and it should be stated no more strongly than the interaction supports. A DNS lookup alone does not
prove that a response was read or that code ran.

---

## Choosing a profile

1. Can you make the server compute something only it could compute, or print something only it could know?
   Use `expression_evaluation.v1` or `command_output_readback.v1`.
2. Can you show a difference between a harmless request and the probing one? Use
   `file_read_differential.v1`.
3. Is the effect invisible in the response? Use `oob_interaction.v1`.
4. Otherwise it is an observation: use `signal.v1` and let the report say *Likely*.

Choosing the weaker profile is always acceptable. Choosing a stronger one than the evidence supports is the
one thing a rule must not do.

## What proof profiles do not protect against

Profiles make honest rules hard to get wrong: they force a control, a baseline, an engine-owned challenge.
They are not a defence against a rule written to mislead — a marker that matches any page, a control pointed
at an unrelated endpoint. That is why rules are reviewed before they enter a catalog, why packs are signed by
a named publisher, and why every finding records the exact rule and version that produced it. See
[packs and distribution](packs.md).
