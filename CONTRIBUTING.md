# Contributing

Thank you for helping improve CDL. Contributions are welcome in three areas: rules, documentation, and
the language itself.

## Rules

A rule is accepted when it is correct, honest about what it proves, and safe to run.

**Before you open a pull request**

1. One rule per file, named after the rule's local id.
2. `ggsec rules validate <file>` reports no errors and no warnings.
3. Add fixtures next to the rule and make sure both pass:
   - one where the weakness is present, with the expected outcome;
   - one where the target is patched or only echoes its input, expecting `NoMatch`.
4. Say in the pull request how the rule was checked against a real instance (product, version, lab setup).

**What reviewers look for**

- **The proof matches the evidence.** A text match is `signal.v1`. A confirming profile needs a real
  baseline or control — not one pointed at an unrelated endpoint — and a marker that matches the specific
  output, not any page. See [proof profiles](docs/proof-profiles.md).
- **`side_effects` is truthful**, and anything the rule creates is removed in `cleanup`, addressed by an id
  the rule generated or extracted.
- **The rule does the least that proves the point.** No destructive requests, nothing that degrades the
  target, no reading of data beyond what demonstrates the finding.
- **`preconditions` keep a product-specific rule away from unrelated targets.**
- **No credentials or secrets** in the rule, and no hard-coded hosts — paths are relative to the target.
- **`info` is written for the reader of a report**: what was observed, why it matters, how to fix it, with
  references.

**Namespaces.** Use your own namespace for rules you maintain yourself. `cortex.official` and
`cortex.community` are assigned by the maintainers when a rule is accepted into those catalogs.

## Documentation

Corrections and clarifications are welcome. If a document and the engine disagree, please open an issue
that shows the rule, the command and the output.

## The language

Changes to the language are discussed in an issue before any pull request. Describe the check you cannot
express today and why existing features do not cover it. Additions must keep every existing rule valid.

## Security

Please do not report security problems in the engine or in pack verification through public issues. Write
to <research@ggsec.de> first, so a fix can be prepared before the details are public.
