# comfy-sync-runner — compute-only GHA runner for comfyui_backend

This repository is a **compute-only runner**: it hosts GitHub Actions
workflows that execute the scheduled jobs of the private
**[beulahkemp/comfyui_backend](https://github.com/beulahkemp/comfyui_backend)**
repo (a ComfyUI SaaS backend with a GPU-market pricing pipeline). It
intentionally contains *nothing but workflow definitions and a tiny
`state/` dir* — no source code, no data, no credentials. (Public repos
get unlimited Actions minutes, which is why the compute moved here when
the source account's Actions became billing-blocked.)

## Workflows

| Workflow | Trigger | Purpose |
|---|---|---|
| `sync.yml` | hourly (:07) / dispatch | The pricing data refresh: checks out the **private** repo via the `GH_PAT` secret, runs the headless sync (`seed → price-history replay → web3 listings → pricing channels`), then race-safely commits the two data files (`model-catalog.json` + `price-history.jsonl`) back to the private repo with `[skip ci]`. Keeps `state/last-run.json` fresh (≤1 commit/day — resets GHA's 60-day schedule-inactivity timer). |
| `kill-dog.yml` | every 10 min / dispatch | Cloud-side GPU pod kill watchdog (L2): if the kill-dog gist heartbeat is stale (the owning dev server is gone) or a pod breached its TTL, terminate the RunPod pod. Logic lives in the private repo (`scripts/kill_dog.py`) so L1 (local daemon) and L2 share ONE implementation. |
| `smoke.yml` | push / dispatch | Connectivity self-test: secrets present, private checkout, gist roundtrip, RunPod auth ping. |

## Security posture

- All credentials (the GitHub PAT used for private checkouts/pushes and
  gist state writes, the RunPod API key, the gist id) live as
  **encrypted GitHub Actions secrets** of this repo — never in any file
  here.
- This repo is public, so **run logs are public**: registered secret
  values are masked automatically by GitHub's log redaction wherever
  they appear. The workflows additionally never interpolate secrets
  into echoed commands.
- **No `pull_request` triggers** exist on the pipeline workflows, so
  fork PRs can never access secrets. Only org admins can push here.
- Every private-repo access authenticates with the `GH_PAT` secret;
  this repo's own default `GITHUB_TOKEN` cannot read or write anything
  private (workflows declare `permissions: {}`).
- What runs here: the private repo's own code, checked out at run time
  on an ephemeral GitHub-hosted runner. Nothing sensitive is committed
  back to this repo (the state file carries a date and a purpose line
  only).

## Triaging a failed run

- **sync failures** — read the channel-coverage lines in the log: the
  sandbox-only page-reader channels are *expected* to fail here
  (`RUNNER-LIMITED`); web3 listings + raw-HTML channels are the ones
  that matter. A 401 on checkout/push means the `GH_PAT` secret
  expired. A rejected push means a parallel session pushed to the
  private repo's main mid-run — the run fails loudly by design and the
  next hourly run re-syncs.
- **kill-dog failures** — check the gist (id in secrets) comments for
  recent kill records and `state.json` for the heartbeat; a failing
  sweep retries in 10 minutes.

Operation runbooks live in the private repo (`docs/RUNBOOK.md`
§Org runner + §Kill-dog).
