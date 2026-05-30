# busbar-actions/sf-snapshot

Create a Salesforce scratch-org snapshot (`OrgSnapshot`) on the DevHub. The companion to `sf-org-create --snapshot <name>` for the **golden-snapshot pattern**: build a snapshot once (with `busbar-broker-sf` + trust CMDT preinstalled), then create scratch orgs from it for every CI run.

## What it does

1. Authenticates to the DevHub via [`busbar-auth`](../../busbar-extensions/crates/busbar-auth).
2. Inserts `OrgSnapshot` (Data API) pointing at the source scratch.
3. Polls until `Status = Active` (or `Error`).
4. Emits the snapshot record id for downstream reference.

## Inputs

| Input | Default | Description |
|---|---|---|
| `name` (required) | — | Snapshot name. Must be unique in the DevHub. |
| `source-org` (required) | — | 15- or 18-char id of the source scratch org (the `scratch-org-id` output from `sf-org-create`). |
| `description` | `` | Optional description. |
| `content` | `` | Optional content selector (usually `metadatadata`). |
| `poll-timeout-secs` | `3600` | Snapshots can take 30+ minutes for large orgs. |
| `result-output` | `.busbar/snapshot-result.json` | Where to write the snapshot result JSON. |
| `version` | `latest` | `sf-snapshot` release tag. |
| `binary-repo` | `busbar-actions/actions-dist` | Where to fetch the binary. |

## Outputs

| Output | Description |
|---|---|
| `snapshot-id` | `OrgSnapshot` record id. Pass to `sf-org-delete --snapshot-id` when retiring. |
| `status` | Final status (`Active` on success). |

## Env vars (from the Environment, per the busbar convention)

- `SF_INSTANCE_URL` — the DevHub.
- For OIDC: `BUSBAR_ECA_CLIENT_ID` (non-secret), `BUSBAR_TOKEN_HANDLER` (default `BBGitHubTokenExchangeHandler`), `BUSBAR_OIDC_AUDIENCE`. The runner provides `ACTIONS_ID_TOKEN_REQUEST_*` with `id-token: write`.
- Local-dev fallback: `SF_ACCESS_TOKEN` + `SF_INSTANCE_URL`.

## Example: golden-snapshot build workflow

```yaml
permissions:
  contents: read
  id-token: write

jobs:
  build-golden:
    runs-on: ubuntu-latest
    environment: devhub-pbo-scratch
    steps:
      - uses: busbar-actions/sf-org-create@v1
        id: org
        with:
          org-name: golden-source
          admin-email: ci@example.com
          duration-days: 1

      - name: Install busbar-broker-sf + trust CMDT into the new org
        env:
          SF_INSTANCE_URL: dynamic # read from credentials file
        run: |
          export SF_ACCESS_TOKEN=$(jq -r .credentials.access_token "${{ steps.org.outputs.credentials-path }}")
          export SF_INSTANCE_URL=$(jq -r .credentials.instance_url "${{ steps.org.outputs.credentials-path }}")
          # … install broker package, deploy trust rules …

      - uses: busbar-actions/sf-snapshot@v1
        with:
          name: golden-busbar
          source-org: ${{ steps.org.outputs.scratch-org-id }}

      - uses: busbar-actions/sf-org-delete@v1
        if: always()
        with:
          scratch-org-info-id: ${{ steps.org.outputs.scratch-org-info-id }}
```
