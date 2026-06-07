# LibreELEC Actions — Work Notes

## Context

Fork: `heitbaum/actions` → upstream: `LibreELEC/actions`

---

## Pending

### `tools-clean-img.yml` trigger investigation — RESOLVED (2026-06-30)

Triggered by test.libreelec.tv storage full (3TB). chewitt: "keep 3-months and that's enough".
Findings:
1. `NIGHTLY_UPLOAD_PATH` = `/var/www/test` — same tree `tools-cleanup-images.yml` hardcodes.
2. `prune-archive.py -k N` — N is **DAYS, not a file/build count**. Keeps EVERY nightly `.img.gz`
   for ALL devices within N days untouched; only files OLDER than N days get thinned to 1 per
   ISO week per device. Only touches `*.img.gz` (not `.tar`/`.ova`/`.sha256`). `-k 200` ≈ 6.6
   months of full daily retention → why storage filled (Allwinner/Amlogic/Rockchip × many uboot
   variants × daily × 200d).
3. The two workflows are contradictory in policy but it's moot: only `tools-cleanup-images.yml`
   runs (daily cron). `tools-clean-img.yml` trigger `workflow_run: [build-images]` is dead —
   confirmed no `build-images` workflow exists in repo or upstream; only manual dispatch fires it.

Actions taken (branch `cleanup-images-retention`, committed locally, NOT pushed):
- `tools-cleanup-images.yml`: keep the 200-day baseline for all projects; add a targeted
  `-k 90` (3-month) pass for Allwinner, Amlogic, Rockchip only (chewitt: "we should not do
  this at every project"). The 90-day pass loops over the three on the host with a
  dir-existence guard so a project absent on a branch is skipped, not failed. Both 12.2 + 13.0.
- `prune-archive.py` (release-scripts submodule): patched to delete companion `.sha256` when
  purging `.img.gz`. **Merged upstream as `ca49832`**; submodule pointer bumped `354fd6e → ca49832`
  in the same commit.

REMAINING: push `cleanup-images-retention` and open the PR to upstream (only when explicitly
asked — do not push without permission).

### SPDX/licensing in workflow YAML files

Scripts are done. `.github/workflows/*.yml` and `.github/actions/` files not yet covered.

### Build host /source/ cleanup

Prune source tarballs (`.tgz`, `.sha256`, `.url`) on self-hosted build servers, keeping
only files referenced by stable and master branches — disk space management.

---

## Known issues

- **Deprecated GraphQL label node IDs** in `tools-update-cacert-pem-certificate-bundle.yml`
  "Add labels to Pull Request" step. The `labels { nodes { id } }` query returns legacy
  base64 IDs (`MDU6...`); `addLabelsToLabelable` still works but warns. Fix: switch to
  REST API (`gh api repos/.../issues/{pr_number}/labels --method POST`) which takes label
  names not IDs — needs testing in org context before adopting.
  **Note: `gh pr edit --add-label` does NOT work in an organisation context.**
  Current behaviour is warning-only; mutation still succeeds.

- **actionlint false positive** on `client-id` in `actions/create-github-app-token@v3`
  (stale bundled metadata). Will not be upstreamed — false-positive is dormant in practice
  as reviewdog only triggers when those lines are modified.
  Upstream fix: `rhysd/actionlint#668`

---

## Slack notifications

All notification steps use `slackapi/slack-github-action@v3` with `webhook-type: incoming-webhook`.
Payloads are pure Block Kit — no legacy `attachments`. Structure:

```yaml
payload: |
  username: "ADDON Bot"
  icon_emoji: ":ghost:"
  channel: "${{ secrets.ADDONS_REPO_WEBHOOK_CHANNEL }}"
  blocks:
    - type: header
      text: {type: plain_text, text: ":red_circle: Build failures", emoji: true}
    - type: section
      text: {type: mrkdwn, text: "${{ steps.failures.outputs.text }}"}
```

Message types and emojis:
- `:red_circle:` — build failures (all 10 `yml-uses-*` files)
- `:warning:` — addon warnings (`deploy-addons.yml`)
- `:large_green_circle:` — addon updates (`deploy-addons.yml`)

`deploy-addons.yml` flow: `update_addon_repo.sh` runs on the server via SSH and emits a
structured JSON object to stdout (version, architectures, warnings, optional error).
The GHA job captures it, then a Python3 `run:` step formats the Block Kit message text,
and separate `slackapi/slack-github-action@v3` steps send warnings and updates.

`update_addon_repo.sh` no longer sends Slack directly — all notification logic lives in GHA.

---

## Linting notes

- `reviewdog/action-actionlint@v1` only scans `.github/workflows/*.yml`, not
  composite actions under `.github/actions/`.
- shellcheck installed at:
  `C:\Users\RudiHeitbaum\AppData\Local\Microsoft\WinGet\Packages\koalaman.shellcheck_Microsoft.Winget.Source_8wekyb3d8bbwe\shellcheck`

