# satsignal-action

GitHub Action that anchors a build artifact to the BSV blockchain via
[Satsignal](https://satsignal.cloud) and exports the proof as step
outputs.

Use it on the things later disputes will be about: AI eval results,
release manifests, agent-generated code diffs, security scan output,
model cards, deployment approvals. Drop it after the step that
produces the artifact; the next step gets a tamper-evident proof.

## Quick start

```yaml
- name: Anchor eval results
  id: anchor
  uses: Steleet/satsignal-action@v0
  with:
    path: ./eval-results.json
    folder: release-${{ github.run_id }}
    label: ${{ github.sha }}
    api-key: ${{ secrets.SATSIGNAL_API_KEY }}

- name: Echo proof
  run: |
    echo "txid: ${{ steps.anchor.outputs.txid }}"
    echo "proof: ${{ steps.anchor.outputs.proof_url }}"
```

> **Compatibility.** `matter:` is a frozen alias of `folder:`, and the
> `receipt_url` / `bundle_id` outputs are frozen aliases of `proof_url` /
> `proof_id`. Existing workflows keep working byte-for-byte with no
> change; new ones should use `folder:` / `proof_url`. Since v0.4.0 the
> action speaks the canonical Satsignal wire vocabulary (sends
> `folder_slug`, reads `proof_id` / `proof_url`); see
> [Server compatibility](#server-compatibility) if you target an older
> self-hosted server.

The action computes `sha256(path)` locally and sends only the hash to
Satsignal. The file's bytes never leave the runner.

## Inputs

| name | required | default | notes |
|---|---|---|---|
| `path` | yes | — | File to anchor, relative to workspace |
| `api-key` | yes | — | Satsignal Bearer token (use a secret) |
| `folder` | no | `github-actions`¹ | Folder slug in your workspace |
| `matter` | no | — | **Legacy** alias of `folder` — kept working forever |
| `label` | no | `""` | Short label rendered on the proof |
| `force-new` | no | `false` | Re-anchor even if sha was seen before |
| `api-base` | no | `https://app.satsignal.cloud` | Override only for self-hosted |
| `fail-on-error` | no | `true` | Record + continue if `false` |
| `provenance` | no | `false` | Wrap the digest in a `satsignal.provenance.v1` manifest with typed-authority metadata from the GitHub Actions env (see below). |

¹ When neither `folder` nor `matter` is set the slug defaults to
`github-actions` (unchanged from before). `folder` takes precedence over
`matter`. Setting **both** to *different* non-empty values fails the step
loudly (`folder and matter are aliases and must not be set to different
values; use folder`) — this mirrors the Satsignal API's
`conflicting_alias` behavior. Equal values are accepted.

## Outputs

| name | example | notes |
|---|---|---|
| `proof_id` | `47f56fdd93ac474b` | Use with `/bundle/<id>.mbnt` to download |
| `bundle_id` | `47f56fdd93ac474b` | **Legacy** alias of `proof_id` — kept forever |
| `txid` | `97108b01212a…` | On-chain BSV transaction id |
| `proof_url` | `https://app.satsignal.cloud/w/…/r/…` | Proof page |
| `receipt_url` | `https://app.satsignal.cloud/w/…/r/…` | **Legacy** alias of `proof_url` — kept forever |
| `bundle_url` | `https://app.satsignal.cloud/bundle/<id>.mbnt` | Bundle download (Bearer required) |
| `sha256` | `fbe7c628…` | sha256 of the anchored file |
| `duplicate` | `false` | `true` if the API returned a prior anchor |

`proof_id` / `proof_url` always carry the same values as the legacy
`bundle_id` / `receipt_url`. The action reads the API's canonical
`proof_*` response keys (the only keys current servers emit) and
transparently falls back to the legacy `receipt_url` / `bundle_id` keys
against older / self-hosted servers.

## Server compatibility

As of v0.4.0 the action sends the canonical `folder_slug` request key
(the Satsignal API's 2026-06 vocabulary sunset: responses emit canonical
keys only; requests still accept legacy keys as silent aliases). If you
point `api-base` at a self-hosted Satsignal server *older* than the
sunset — one that only accepts `matter_slug` — pin
`Steleet/satsignal-action@v0.3.0`, which sends the legacy key.

## Proof-on-PR pattern

```yaml
- name: Anchor release manifest
  id: anchor
  uses: Steleet/satsignal-action@v0
  with:
    path: ./release-manifest.json
    folder: deploys
    label: ${{ github.ref_name }}
    api-key: ${{ secrets.SATSIGNAL_API_KEY }}

- name: Comment on PR
  if: github.event_name == 'pull_request'
  uses: actions/github-script@v7
  with:
    script: |
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: `Release manifest anchored on BSV: [\`${{ steps.anchor.outputs.txid }}\`](${{ steps.anchor.outputs.proof_url }})`,
      })
```

## Provenance mode (typed-authority)

By default the action anchors a bare file digest. Set `provenance:
'true'` to instead build a `satsignal.provenance.v1` manifest with a
typed-authority block (Phase-4 / 2026-05-20) and anchor that
manifest. The file digest still rides as `subject.digest` inside the
manifest — bytes never leave the runner.

```yaml
- name: Anchor with provenance
  uses: Steleet/satsignal-action@v0
  with:
    path: ./eval-results.json
    folder: ai-evals
    provenance: 'true'                          # opt in
    api-key: ${{ secrets.SATSIGNAL_API_KEY }}
```

### What the manifest captures

The action populates these fields from the GitHub Actions environment.
Empty env vars are omitted; the notary's strict validator rejects empty
ids.

| Manifest field | Source | Example |
|---|---|---|
| `source` | `{type: "github", id: $GITHUB_REPOSITORY}` | `acme/widgets` |
| `subject.digest` | `sha256(path)` | `sha256:fbe7…` |
| `authority` | `{type: "organization", id: "github:$GITHUB_REPOSITORY_OWNER", name: …}` | `github:acme` |
| `organization` | `{type: "company", id: "github:$GITHUB_REPOSITORY_OWNER"}` | `github:acme` |
| `principal` | `{type: "service-account", id: "github:$GITHUB_ACTOR", name: …}` | `github:deploy-bot[bot]` |
| `agent` | `{type: "ci-runner", id: "github-actions/$GITHUB_RUNNER_NAME"}` | `github-actions/Hosted-12` |
| `run_scope` | `{type: "workflow", id: $GITHUB_WORKFLOW_REF, environment: $GITHUB_REF_NAME}` | `acme/widgets/.github/workflows/release.yml@refs/tags/v1.4.2` |
| `privacy` | `{onchain_mode: "hash_only"}` | — |

`run_scope.id` uses `$GITHUB_WORKFLOW_REF` (GitHub's canonical form
that already pins both workflow path and ref), falling back to
`$GITHUB_WORKFLOW@$GITHUB_WORKFLOW_SHA` when only the legacy env var is
set.

### Verifying a typed-authority anchor

The output set is identical to standard mode (`proof_id`, `txid`,
`proof_url`, etc.). The full canonical manifest ships inside the
`.mbnt` bundle and is byte-exactly re-derivable offline — anyone with
the bundle can JCS-canonicalize the manifest, sha256 it, and walk
that hash to the on-chain transaction without calling any Satsignal
API. See [provenance-v1.md](https://proof.satsignal.cloud/spec-provenance)
§8 for the validator surface.

### Server requirements

Requires Phase-4 (2026-05-20) Satsignal notary or later. Older /
self-hosted servers will reject the typed-authority fields as
`unknown_field`. Keep `provenance: 'false'` (the default) for workflows
that target older deployments.

## Dedup behavior

By default the Satsignal API deduplicates: re-anchoring the same
`sha256_hex` in the same folder returns the prior proof without
broadcasting a new transaction. The action's `duplicate` output is
`true` in that case.

Set `force-new: true` to anchor again (e.g. to refresh the chain
timestamp on a re-release).

## Verification

The proof is independent of Satsignal — anyone can fetch the
`.mbnt` bundle and verify the on-chain transaction directly against
BSV. See [bundle-v1.md](https://proof.satsignal.cloud/spec-bundle)
for the format, or use the
[`satsignal-cli`](https://github.com/Steleet/satsignal-cli) /
[`satsignal-mcp`](https://github.com/Steleet/satsignal-mcp) packages
for one-line verifier scripts.

## Preconditions

- **API key** with the `anchors:create` scope. Generate at
  <https://app.satsignal.cloud>, store as a repository secret
  (`SATSIGNAL_API_KEY`), never hardcode in a workflow.
- **Folder exists.** `inbox` is auto-provisioned for every workspace
  and works out of the box; any other slug must be created first in
  your workspace at <https://app.satsignal.cloud>. The action 404s
  with `folder_not_found` if the slug doesn't resolve in your
  workspace.

## License

MIT.
