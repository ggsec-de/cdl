# Community rules

The rules in this directory were converted automatically to CDL from the
[Nuclei Templates](https://github.com/projectdiscovery/nuclei-templates) project. They are redistributed under
that project's MIT License — see [LICENSE](LICENSE) in this directory.

## What to expect from them

- **Every rule here is `signal.v1`.** A converted matcher has no baseline, control or challenge, so a hit is
  an observation and is reported as *Likely*, never *Confirmed*. See
  [proof profiles](../../docs/proof-profiles.md).
- **They are read-only and anonymous.** Each rule declares `side_effects: read_only` and `auth: anonymous`.
- **They were converted, not rewritten.** Only templates that could be expressed faithfully in CDL were
  converted; templates that need features CDL does not have were left out rather than approximated.
  Severity and descriptions come from the source template.
- **Their ids are in the `cortex.community` namespace** and carry the tags `nuclei` and `auto-ported`.

Treat a hit from this directory as a lead to verify. Where a weakness matters to you, a rule with a
confirming proof — see [`rules/official/`](../official/) for the pattern — is the better tool.

## Checking them

```bash
ggsec rules validate rules/community/
```

## Improving them

A converted rule that produces false positives, or one that could be upgraded to a confirming proof, is a
welcome contribution — see [CONTRIBUTING.md](../../CONTRIBUTING.md). An upgraded rule belongs in your own
namespace or, after review, in `rules/official/`.
