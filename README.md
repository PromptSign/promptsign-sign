# promptsign-sign

Sign AI instruction files, meaning skills, agent definitions, `CLAUDE.md`, and
`AGENTS.md`, from GitHub Actions with the workflow's own verified identity. No
signing key, no secret to store, nothing to rotate.

```yaml
permissions:
  id-token: write        # required; the action cannot grant this to itself
  contents: read

steps:
  - uses: actions/checkout@v4
  - uses: PromptSign/promptsign-sign@v1
    with:
      path: skills/my-skill
      version: ${{ github.ref_name }}
```

That produces `skills/my-skill/.promptsign/bundle.json`, a detached signature
carrying the identity of the workflow that made it (a single file gets a
`<file>.psig.json` sidecar instead). Anyone can check it, offline:

```sh
promptsign verify skills/my-skill
```

## Inputs

| Input | Default | What it does |
|---|---|---|
| `path` | *required* | The file or directory to sign. A directory is signed as one manifest, scripts included. |
| `version` | none | The version recorded in the manifest, usually the release being cut (`${{ github.ref_name }}`, or `1.4.0`). Verifiers report it, so a consumer can tell *which* release a copy came from. An unversioned signature still proves origin, but not which build. |
| `name` | basename of `path` | The name recorded in the manifest. |
| `kind` | none | What this is: `skill`, `plugin`, `agent`, `instructions`. |
| `cli-version` | `v0.3.0` | Which [promptsign release](https://github.com/PromptSign/promptsign-cli/releases) to install. Pin it; don't track `latest`. |
| `embed` | `false` | Write the signature into the file's own frontmatter instead of a sidecar, so it travels with the file. Single Markdown files only. |
| `args` | none | Anything else passed through to `promptsign sign`. Don't repeat `--name`, `--version`, or `--kind` here. Those have their own inputs, and passing both sends the flag twice. |

## Outputs

| Output | What it is |
|---|---|
| `identity` | The certificate identity the signature carries. This is the string a consumer pins in policy. |
| `version` | The version recorded in the signed manifest. |
| `bundle` | Path to the signature that was written. With `embed: true` this is the signed file itself, since the signature lives in its frontmatter. |

The action writes a job summary naming the artifact, its version and the signing
identity, and it re-verifies its own output before finishing: a signature that
does not verify fails the step instead of being left behind.

## What identity you get

Fulcio issues a short-lived certificate to the workflow itself, so the signature
names the thing that produced it:

```
https://github.com/OWNER/REPO/.github/workflows/publish.yml@refs/heads/main
```

with issuer `https://token.actions.githubusercontent.com`. That identity is the
workflow file's path and the ref it ran from. Rename the file or run it from a
different branch and the identity changes, which is the point: it describes what
signed.

A consumer pins it in policy with a glob:

```json
{ "identity": "https://github.com/OWNER/*/.github/workflows/*",
  "issuer": "https://token.actions.githubusercontent.com" }
```

## Two things to know

**Signatures are public.** The identity goes into Rekor, a public append-only
transparency log. Signing from a private repository publishes that repository's
name and workflow path, and there's no taking it back.

**Fork pull requests can't sign.** GitHub doesn't grant `id-token: write` to
workflows triggered from forks. That's a deliberate boundary rather than a bug
to work around, so sign on merge, or on release.

## Pinning

`@v1` follows the latest `v1.x` release, so fixes arrive without editing your
workflow. It is a moving tag, which means whoever can push to this repository
decides what runs in your CI, and those runs sign under your identity. Two
tighter options, in increasing order of strength:

```yaml
  - uses: PromptSign/promptsign-sign@v1.0.0                        # exact release
  - uses: PromptSign/promptsign-sign@8b0e6f2...  # v1.0.0          # exact commit
```

A tag can be moved and a commit SHA cannot, so the SHA is the one to use when
this action signs artifacts other people rely on. Dependabot and Renovate both
update SHA pins and keep the version in the trailing comment current, so you
give up less than it looks.

## Supply chain

The action downloads the release archive and verifies it against the release's
`SHA256SUMS` before running it. Every archive is also published with a detached
PromptSign signature (`.psig.json`) for anyone bootstrapping trust by hand. See
the [installation notes](https://github.com/PromptSign/promptsign-cli).

Trust material and TOFU pins are kept under `$RUNNER_TEMP` for the duration of
the job, so a run leaves nothing behind on a self-hosted runner and cannot be
affected by what an earlier job pinned.

Linux runners only, on x86_64 or arm64. macOS and Windows runners can call the
CLI itself, and the [release page](https://github.com/PromptSign/promptsign-cli/releases) has binaries for both.

## Learn more

Pair this with [`promptsign-verify`](https://github.com/PromptSign/promptsign-verify)
to gate pull requests on the signature this action produces. Full docs, including the
npm SDK and the Claude Code plugin, are on the [Integrate](https://promptsign.ai/integrate?c=actions-listing) page.

## License

Apache-2.0.
