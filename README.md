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

> **Legacy spelling still works.** `matter:` is a frozen alias of
> `folder:`, and `receipt_url` / `bundle_id` are frozen aliases of
> `proof_url` / `proof_id`. Existing workflows keep working byte-for-byte
> with no change; new ones should prefer `folder:` / `proof_url`.

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
`bundle_id` / `receipt_url`. The action reads the API's new `proof_*`
response keys when present and transparently falls back to the legacy
`receipt_url` / `bundle_id` keys against older / self-hosted servers.

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
  and works out of the box; any other slug must be created first at
  <https://app.satsignal.cloud/w/{workspace}/matters>. The action 404s
  with `matter_not_found` if the slug doesn't resolve in your
  workspace.

## License

MIT.
