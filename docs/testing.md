# Testing rules offline

Every CDL rule can be checked without a target and without a network:

| Command | What it does |
|---|---|
| `ggsec rules validate <file-or-directory>` | Checks structure, references, budgets and capabilities. |
| `ggsec rules inspect <file>` | Prints the request plan, auth, proof and cleanup. Sends nothing. |
| `ggsec rules test <rule> --fixture <file.json>` | Runs the rule against canned responses and prints the verdict. |

`validate` exits non-zero when any rule has an error, so it can gate a pull request on its own.

## Validation messages

Each problem is reported with a code, a message, and the file position:

```
error ref.forward: forward reference to step 'read' (not yet executed here)  (my-rule.yaml:24:7 · steps/0/when)
```

Errors stop the rule from running at all. Warnings (for example an `impact` that does not match the proof
profile) do not, but should be fixed.

## Fixtures

A fixture is a small JSON file that answers the rule's requests. `rules test` routes every request the rule
sends to the first matching entry and returns its response. Nothing leaves the process.

```json
{
  "responses": [
    { "method": "GET",  "path": "/status",  "status": 200, "headers": { "Server": "demo/1.0" }, "body": "ok" },
    { "pathContains": "report=",            "status": 200, "body": "report contents" },
    { "method": "POST", "path": "/api", "bodyContains": "\"mode\":\"b\"", "status": 500, "body": "error" }
  ],
  "default": { "status": 404, "body": "not found" }
}
```

**Matching a request.** An entry matches when every key it gives matches:

| Key | Matches |
|---|---|
| `method` | the HTTP method (any method when omitted) |
| `path` | the request path, exactly, without the query string |
| `pathContains` | a substring of the path including the query string |
| `bodyContains` | a substring of the request body — tells apart two requests to the same path |

Entries are tried in order and the first match wins, so put specific entries before general ones. A request
that matches nothing gets `default`, or a 404 when there is no `default`.

**The response.** `status` (default 200), `headers`, and `body`.

**Simulating failures.** `"transportState": "timeout"`, `"network_error"` or `"blocked"` makes the request
fail in that way; `"truncated"` returns the body flagged as cut off. Use these to check that a rule ends
`Inconclusive` instead of reporting a clean `NoMatch` when the target misbehaves.

**Dynamic responses.** Confirming proofs use values the engine picks at run time, which a static body cannot
predict. Four directives stand in for a target that reacts to them:

| Directive | The fixture responds with | For |
|---|---|---|
| `"reflectBody": true` | the request body, unchanged | echo and control endpoints |
| `"evalProduct": true` | `{"result": <a*b>}` for the `a*b` found in the request body | `expression_evaluation.v1` |
| `"commandEcho": true` | the challenge token followed by a line of command-style output | `command_output_readback.v1` |
| top-level `"oob": { "deliver": true, "protocol": "http" }` | a callback on the run's address | `oob_interaction.v1` |

`commandEcho` looks for the token in the current request and then in earlier requests of the same run, so a
rule that triggers a command in one step and reads its output in a later one can be tested too.

Fixtures are stateless: an entry returns the same response every time. To test a flow whose responses change
(create, then read, then delete), give each request its own path or method, as
[`examples/fixtures/resource-lifecycle.json`](../examples/fixtures/resource-lifecycle.json) does.

## Reading the output

```
Rule     : example:resource-lifecycle v1.0.0  (Signal)
Fixture  : examples/fixtures/resource-lifecycle.json  (seed 1337)

Transcript:
  #1  create         POST   /demo/notes  -> 201 Success
  #2  read           GET    /demo/notes/n-1001  -> 200 Success
  #3  remove         DELETE /demo/notes/n-1001  -> 204 Success  · cleanup

Verdict  : execution=Completed  outcome=Signal  cleanup=Succeeded
```

The transcript lists every request in the order it was sent, with the step that sent it. The verdict line
shows the three result values described in the
[language reference](language-reference.md#3-the-result-model). When the outcome is not what you expected, the
`Reason` line says which condition decided it.

Random values are reproducible in a test run: the same `--seed <n>` gives the same nonces and challenges.

## Assertions for CI

`--expect <NoMatch|Signal|Confirmed|Undetermined>` turns a test into an assertion; the command exits non-zero
when the outcome differs.

```bash
ggsec rules test examples/challenge-arithmetic.yaml --fixture examples/fixtures/challenge-arithmetic.json --expect Confirmed
```

```bash
ggsec rules test examples/challenge-arithmetic.yaml --fixture examples/fixtures/challenge-arithmetic.echo.json --expect NoMatch
```

**Test the negative case too.** A rule is only as good as its ability to stay quiet. For every rule, keep at
least two fixtures: one where the weakness is present, and one where the target is patched or merely repeats
its input. The second is the one that catches a proof that is too easy to satisfy.

## What a fixture cannot tell you

A fixture shows that the rule's logic is right for the responses you wrote. It does not show that a real
target answers that way. Before a rule is published it should also be run against a real, deliberately
vulnerable instance in a lab — and against a patched one.
