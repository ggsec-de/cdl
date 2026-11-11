# Packs and distribution

A single rule is a file you can load from a directory. To distribute rules to other people — and to let them
check where the rules came from — CDL rules are shipped as **signed packs**.

- [What a pack is](#what-a-pack-is)
- [Trust](#trust)
- [Installing and updating](#installing-and-updating)
- [Publishing your own pack](#publishing-your-own-pack)
- [Feeds](#feeds)
- [What signing does and does not mean](#what-signing-does-and-does-not-mean)

## What a pack is

A pack is a ZIP archive with the extension `.cortex-rules`:

```
manifest.json      what the pack contains and who published it
manifest.sig       the publisher's signature over manifest.json
rules/*.yaml       the rules
assets/…           optional text assets listed in the manifest
```

A pack contains data only. There is no executable code in it, and a rule cannot include other files.

`manifest.json`:

| Field | Meaning |
|---|---|
| `format_version` | `1` |
| `publisher` | The name of the publisher. |
| `pack_id` | The pack's identifier, `[a-z0-9][a-z0-9._-]*`. |
| `pack_version` | SemVer, for people. |
| `release_sequence` | A number that only ever increases with each release, for machines. |
| `engine_api_min`, `engine_api_max` | The engine API versions the pack works with. |
| `capabilities` | The capabilities its rules need. |
| `issued_at` | When the pack was built. |
| `key_id` | The id of the key that signed it. |
| `files` | Every file in the pack: `path`, `size`, `sha256`. |

The signature is ECDSA P-256 with SHA-256 over the exact bytes of `manifest.json`. Because the manifest lists
the SHA-256 of every file, changing one byte of any rule breaks either a file hash or the signature, and the
pack is rejected.

## Trust

A pack is accepted only when it is signed by a key the user has chosen to trust. Trust is a file named
`<anything>.trust.json` in the `trust/` directory of the rule store:

```json
{
  "publisher": "ACME",
  "trust": "company",
  "public_key_pem": "-----BEGIN PUBLIC KEY-----\n…\n-----END PUBLIC KEY-----\n"
}
```

- The key is **bound to a publisher name**. A pack whose manifest names a different publisher than the key is
  bound to is rejected, so one publisher cannot sign as another.
- `trust` is `official` or `company`. The publisher name `GGSEC` is reserved and can only be bound to an
  `official` key.
- A pack never brings its own trust. A key or key id inside a pack establishes nothing; only keys already in
  the store count.
- An empty store trusts nobody.

## Installing and updating

```bash
ggsec rules install acme-web-1.0.0.cortex-rules --store ./.cortex-rules
```

```bash
ggsec rules list --store ./.cortex-rules
```

```bash
ggsec rules update https://rules.example/feed --store ./.cortex-rules
```

```bash
ggsec rules rollback acme-web-1.0.0.cortex-rules --store ./.cortex-rules
```

- **`install`** verifies the signature, the publisher binding, every file hash and every rule, and only then
  activates the pack. If anything fails, the previously active pack stays in place.
- **`update`** fetches a signed [feed](#feeds) and installs what is newer. It refuses any pack whose
  `release_sequence` is not higher than the highest one already seen — a feed cannot quietly move you back to
  an older, withdrawn release.
- **`rollback`** is the deliberate way to go back to an earlier pack you still have.
- **`list`** shows the trusted keys, the active packs and the rules they provide.

A scan uses the packs that were active when it started; an update during a scan does not change the rules of
that scan. Rules you keep in a plain directory are loaded with `--rules-dir` and need no signature — they are
your own files, and reports mark them as local.

## Publishing your own pack

Generate a key pair once:

```bash
ggsec rules keygen --out acme
```

This writes `acme.key` (the **private** key — keep it offline and out of version control), `acme.pub`, and
prints the key id.

Put the rules in a directory with a `rules/` subdirectory, validate them, and build the pack:

```bash
ggsec rules validate ./my-pack/rules
```

```bash
ggsec rules pack ./my-pack --publisher ACME --pack-id web-rules --version 1.0.0 --seq 1 --key acme.key --out web-rules-1.0.0.cortex-rules
```

Give your users the `.cortex-rules` file and a trust entry containing `acme.pub`. For each new release,
raise both `--version` and `--seq`; `--seq` must never go down or repeat.

## Feeds

A feed is a directory served over HTTPS that holds the packs plus a signed index:

```
index.json
index.sig
web-rules-1.0.0.cortex-rules
web-rules-1.1.0.cortex-rules
```

Build the index from the packs in the directory rather than writing it by hand:

```bash
ggsec rules build-index ./feed --key acme.key --expires-days 30
```

The index lists, for each pack, its publisher, id, version, release sequence, file name and SHA-256. Clients
verify the index signature, download a pack, compare its hash with the index, and then verify the pack itself
exactly as `install` does — the index only says what is available. Feeds must be HTTPS with a valid
certificate; plain HTTP is refused. After regenerating the index, upload `index.json` and `index.sig`
together.

## What signing does and does not mean

A valid signature tells you **who published a pack and that it has not been altered**. It does not make a
finding more certain: the engine grades a signed official rule and an unsigned local rule in exactly the same
way, from the evidence alone (see [proof profiles](proof-profiles.md)).

What you trust, when you add a publisher's key, is their review process — that their rules do what the
descriptions say and nothing more. Every finding records the rule id, the rule version and the pack it came
from, so a result can always be traced back to the exact rule that produced it.
