> [!WARNING]
> **`busbar-actions` is under heavy active development — expect breaking changes.**
> These repositories are public, but **not ready for use yet** — please don't depend on them.
> A pilot is starting soon: **[star and watch the busbar-actions organization](https://github.com/busbar-actions)** for the launch of Discussions and the pilot announcement.

# busbar-actions/sf-snapshot

Create and rotate a Salesforce scratch-org snapshot (`OrgSnapshot`) on the DevHub via the direct Data API. The companion to `sf-org-create --snapshot <name>` for the **golden-snapshot pattern**: build a snapshot once (with the Busbar managed package + trust CMDT preinstalled), then create scratch orgs from it for every CI run.

## What it does

The `sf-snapshot` binary authenticates to the DevHub via [`busbar-auth`](../../busbar-extensions/crates/busbar-auth) (`session_from_env`), then performs a **zero-downtime rotation** (ported from CumulusCI's `SnapshotManager`). Because `OrgSnapshot` names must be unique and a snapshot takes many minutes to build, a naive "delete then create" leaves a window where the canonical name points at nothing, and a plain create fails on re-run. Instead the binary:

1. Validates the base name (1-13 alphanumeric chars).
2. Builds the new snapshot under a **temp name** (`<base>0`).
3. Cleans up any stale temp snapshot left by a prior interrupted run.
4. Polls the temp snapshot to `Active` (with 1.5x backoff up to 30s).
5. Deletes the **old** active snapshot holding the canonical name (if any).
6. Renames the temp snapshot to the canonical name.

The canonical name therefore always resolves to a working snapshot, and the operation is idempotent / re-runnable — exactly what a scheduled golden-snapshot rebuild needs. The binary writes the result JSON, the `GITHUB_OUTPUT` values, a job-summary table and a notice annotation.

> A CLI-only `--no-rotate` escape hatch creates directly under the canonical name (failing if it already exists). It is **not** exposed by the action; the action always rotates.

## Usage

```yaml
permissions:
  contents: read
  id-token: write   # required for the OIDC → Salesforce token exchange

jobs:
  build-golden:
    runs-on: ubuntu-latest
    environment: devhub-pbo-scratch   # supplies BUSBAR_* via the Environment
    steps:
      - uses: busbar-actions/sf-org-create@v1
        id: org
        with:
          target-instance: ${{ vars.SF_INSTANCE_URL }}   # Busbar-equipped DevHub (busbar-pilot-demo2)
          org-name: golden-source
          admin-email: ci@example.com
          duration-days: 1

      # … install the Busbar managed package + deploy trust CMDT into the new org …

      - uses: busbar-actions/sf-snapshot@v1
        id: snap
        with:
          target-instance: ${{ vars.SF_INSTANCE_URL }}   # Busbar-equipped DevHub (busbar-pilot-demo2)
          name: goldenbusbar
          source-org: ${{ steps.org.outputs.scratch-org-id }}

      - uses: busbar-actions/sf-org-delete@v1
        if: always()
        with:
          target-instance: ${{ vars.SF_INSTANCE_URL }}
          scratch-org-info-id: ${{ steps.org.outputs.scratch-org-info-id }}
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `target-instance` | no¹ | `` | **PRIMARY OIDC path.** Instance URL of the Busbar-equipped DevHub (e.g. `busbar-pilot-demo2`) to self-mint a short-lived token against. Maps to `SF_INSTANCE_URL`. |
| `name` | yes | — | Canonical snapshot name. Must be 1-13 alphanumeric chars and unique within the DevHub. |
| `source-org` | yes | — | 15- or 18-char id of the source scratch org (the `ScratchOrg` field — i.e. `sf-org-create`'s scratch-org id output). |
| `description` | no | `` | Optional snapshot description. |
| `content` | no | `` | Optional content selector. Empty falls back to the binary default `metadatadata`. |
| `poll-timeout-secs` | no | `3600` | Seconds to wait for the snapshot to reach `Active` (can take 30+ min for large orgs). |
| `result-output` | no | `.busbar/snapshot-result.json` | Path to write the snapshot result JSON. |
| `eca-client-id` | no | `` | Optional OIDC tuning → `ECA_CLIENT_ID`. Baked default. |
| `token-handler` | no | `` | Optional OIDC tuning → `TOKEN_HANDLER_APEX`. Defaults to `BBGitHubTokenExchangeHandler`. |
| `oidc-audience` | no | `` | Optional OIDC tuning → `OIDC_AUDIENCE`. Defaults to the target instance URL. |
| `sf-instance-url` | no | `` | **Optional local-dev/advanced override** of the DevHub instance URL; wins over `target-instance`. |
| `sf-access-token` | no | `` | **Optional local-dev/advanced override only.** A pre-obtained DevHub token; when set the binary skips OIDC self-minting. Leave empty in CI. |
| `version` | no | `latest` | Release tag of the `sf-snapshot` binary. |
| `binary-repo` | no | `busbar-actions/actions-dist` | GitHub repo hosting prebuilt binary releases. |

¹ `target-instance` is required for the default OIDC path (or supply the `sf-instance-url`/`sf-access-token` local-dev override).

## Outputs

| Output | Description |
|---|---|
| `snapshot-id` | `OrgSnapshot` record id. Pass to `sf-org-delete --snapshot-id` when retiring. |
| `status` | Final status (`Active` on success). |
| `snapshot-name` | Canonical snapshot name. |
| `rotated` | `"true"` when zero-downtime rotation was used (always true via the action). |
| `expiration-date` | Snapshot expiration date when provided by Salesforce. |
| `result-path` | Path to the JSON file containing the snapshot result (echoes `result-output`). |

## Auth & permissions — OIDC self-mint (default)

> [!NOTE]
> **DevHub prerequisite.** In-process OIDC → DevHub requires a DevHub that has the
> **Busbar managed package installed and a trust rule** for the calling repo's
> workflow. The pilot DevHub `busbar-pilot-demo2` is set up this way — point
> `target-instance` at it (supplied via the workflow Environment's
> `SF_INSTANCE_URL` variable). You can also use the `sf-instance-url` +
> `sf-access-token` override inputs locally.

This action runs against the **DevHub** and **self-mints** its token: the binary
exchanges the runner's GitHub OIDC id-token for a short-lived Salesforce session
**in-process** via `busbar-auth`, runs the rotation, then revokes + zeroizes the
token at exit. **No Salesforce access token is ever handed to a script, written
to `GITHUB_ENV`, or passed as an action input/output.** There is **no** `org-auth`
handoff step.

- Set **`target-instance`** (the Busbar-equipped DevHub URL, e.g. `busbar-pilot-demo2`) → `SF_INSTANCE_URL`, and grant **`permissions: id-token: write`** so the runner can mint the OIDC id-token. The runner then provides `ACTIONS_ID_TOKEN_REQUEST_URL` / `ACTIONS_ID_TOKEN_REQUEST_TOKEN`.
- OIDC tuning (from the Environment or per-run inputs): `BUSBAR_ECA_CLIENT_ID`/`eca-client-id` (non-secret), `BUSBAR_TOKEN_HANDLER`/`token-handler` (the unqualified token-handler name), `BUSBAR_OIDC_AUDIENCE`/`oidc-audience`.
- **Local-dev/advanced override:** set the `sf-instance-url` + `sf-access-token` inputs to skip the exchange; the handed-in token is used directly (zeroized, not revoked).

**Token lifecycle.** The DevHub token is minted and owned by this binary end-to-end. On success it calls `session.dispose()`, which revokes the OIDC-minted token (`/services/oauth2/revoke`) and zeroizes the credential material on drop. The token is **not** handed off to downstream steps via `GITHUB_ENV`, so no separate job-end cleanup is required for the happy path. (Note: a hard error path exits before `dispose()`, so a minted token may not be revoked on failure — see Observability.)

## Observability

The binary emits its `GITHUB_OUTPUT` values, a `$GITHUB_STEP_SUMMARY` table and a notice annotation through the [`github-actions-ux`](../../busbar-extensions/crates/github-actions-ux) `Reporter`, auto-selecting GitHub vs plain-CLI output. Progress/diagnostic lines (DevHub URL, rotation steps, poll status) are still written with `eprintln!` and the error path uses `github_actions_ux::fail()` rather than `run_outcome` + `RecordingReporter`, so on failure those lines and the final error do not land in the Job Summary. Routing the run through `run_outcome` + a `RecordingReporter` (and disposing the session in the error arm) would make errors always reach the Job Summary and guarantee token revocation on failure.
