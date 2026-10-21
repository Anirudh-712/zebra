# The `next` Branch

`next` is where Zebra maintainers merge changes to run them on real nodes before they reach `main`. Every merge into `next` deploys six nodes from it, so the team can compare how they sync with the stage nodes deployed from `main`.

`next` is a long-lived branch. `main` is merged into it after every change, so it always holds everything on `main` plus the work that has not been promoted yet. When that work has proven itself on the nodes, one pull request promotes it from `next` to `main`.

## How Work Moves Through `next`

1. A pull request targets `next` and merges after its required checks pass.
2. The push to `next` deploys the `next` nodes.
3. After each push to `main`, a pull request merges `main` into `next`.
4. When the work on `next` is ready, a pull request promotes `next` to `main`. It goes through `main`'s usual review and merge queue.

Changes that don't need to run on nodes first still target `main` directly.

## Merge Rules

| Rule | `main` | `next` |
| --- | --- | --- |
| Pull request required | Yes | Yes |
| Approvals | 1 | 0 |
| Required checks | `lint`, `unit-tests`, `test-crates`, `pr-gate-result`, `merge-policy`, `mergefreeze` | `lint`, `unit-tests`, `test-crates`, `pr-gate-result`, `merge-policy` |
| Merge queue | Yes | No |
| Merge method | Merge commit | Merge commit |
| Force pushes and deletion | Blocked | Blocked |

Merging still needs write access to the repository, so external contributors' pull requests are merged by a maintainer.

Pull requests into `next` follow the same title and changelog rules as pull requests into `main`. Their commits reach `main` unchanged when `next` is promoted, and release-plz reads those commits to choose versions, so each one needs a [conventional commit](https://www.conventionalcommits.org/en/v1.0.0/#specification) title and the [change fragments](changelog-guidelines.md) its changes require.

## Keep `next` in Sync With `main`

After each push to `main`, the [`sync-next.yml`](https://github.com/ZcashFoundation/zebra/blob/main/.github/workflows/sync-next.yml) workflow opens a `chore: sync main to next` pull request from `main` into `next`. If one is already open, it picks up the new commits by itself. The workflow opens the pull request with the release app token, because pull requests opened with the workflow's own token don't start the required checks.

Merge the sync pull request with a merge commit, so `next` keeps each commit from `main`. Merge it soon after it opens. The longer `next` falls behind, the larger the next conflict, and the less the comparison with stage means.

### Resolve Sync Conflicts

When `main` and `next` changed the same lines, the workflow logs a warning and the sync pull request shows conflicts. Its head is `main`, so the conflicts can't be resolved on it. Resolve them on a branch cut from `next` instead:

```bash
git fetch origin
git switch -c sync/main-to-next-$(date +%Y%m%d) origin/next
git merge origin/main
```

Resolve the conflicts, commit the merge, then open a pull request into `next`:

```bash
git push -u origin HEAD
gh pr create --base next --title "chore: sync main to next" \
  --body "Resolves the conflicts in the automatic sync pull request."
```

Merge it with a merge commit. Once `next` contains the automatic pull request's head, GitHub marks that pull request as merged. If `main` moved on in the meantime, it stays open with only the newer commits.

## Promote `next` to `main`

Open a pull request from `next` into `main` when the work on `next` is ready to release:

```bash
gh pr create --base main --head next --title "feat: promote next to main"
```

Choose the title's type for the most significant change in the batch, and add `!` if any of its commits is breaking. The pull request needs `main`'s approval and goes through the merge queue like any other. Its merge commit keeps every commit from `next`. The next sync pull request then brings that merge commit back into `next`.

The ruleset on `next` blocks deletion, so `next` survives the merge even if the repository deletes head branches automatically.

## The `next` Nodes

| Property | `next` | `stage`, for comparison |
| --- | --- | --- |
| Deployed on | Push to `next` | Push to `main` |
| GCP project | `zfnd-dev-zebra` | `zfnd-dev-zebra` |
| Nodes | Mainnet and Testnet in `us-east1-b`, `-c`, and `-d` | Same |
| MIG names | `zebrad-next-${network}-${zone-letter}` | `zebrad-main-${network}-${zone-letter}` |
| Stateful disks | `zebrad-cache-next-${network}-${zone-letter}` | `zebrad-cache-main-${network}-${zone-letter}` |
| Reserved addresses | `zebra-next-${network}`, `-secondary`, `-tertiary` | `zebra-${network}`, `-secondary`, `-tertiary` |
| `environment` label and Sentry environment | `next` | `stage` |
| GitHub environment | `next` | `stage` |

A push to `next` deploys only when it changes Rust code, dependencies, Docker files, or the deploy workflows, the same filter that applies to `main`. Each deploy waits for the nodes to become healthy, like stage.

`next` doesn't run the GCP integration tests, so it never creates cache images of its own. Its first deploy seeds each disk from the latest `main` cache image, and later deploys keep the disk. The daily cleanup workflow never deletes `zebrad-cache-*` disks or the instances of a MIG, so it leaves the `next` nodes alone.

Six more nodes double stage's node cost: six spot VMs, six 400 GB `pd-balanced` disks, and six reserved addresses.

A `workflow_dispatch` run can also redeploy one `next` node. Select the `next` environment, as in the [runbook](gcp-deployment-operations.md#recover-a-corrupted-cache-disk).

## Compare `next` With `main`

Both node sets run in the same project, zones, and machine type, with the same health check, and start from the same cache images. Differences in how they sync come from the code, peers, and uptime. To list the pull requests on `next` that `main` doesn't have yet:

```bash
git fetch origin
git log --first-parent --oneline origin/main..origin/next
```

In the `zfnd/zebrad` Sentry project, compare the `stage` and `next` environments. CI integration-test nodes on `main` also report as `stage`, so filter on `server.address:zebrad-*` to keep only the deployed nodes, as the stall alerts do.

In Cloud Logging for `zfnd-dev-zebra`, every node logs its estimated progress to the chain tip about once a minute, with `sync_percent` and `current_height`. Compare a `next` node with the stage node in the same zone and network, for example `zebrad-next-mainnet-b` with `zebrad-main-mainnet-b`.

Each deploy restarts the nodes it changes, so compare nodes after both have run for a similar time since their last deploy. To compare syncing from the same starting point, recover both disks from the same cache image with the [corrupted cache disk recipe](gcp-deployment-operations.md#recover-a-corrupted-cache-disk).

A pull request that changes the state database format upgrades the `next` disks when it deploys, and reverting it on `next` doesn't undo that. Within the same major format version, the reverted code opens the upgraded state on a best-effort basis. A major format change keeps the state in a new directory, so the reverted code goes back to the older directory on the disk, which may be stale or empty. The [state database upgrade rules](state-db-upgrades.md) explain both cases. To start clean, recover the `next` disks from a `main` cache image.

## One-Time Setup

These settings live outside git, so a maintainer applies them once before the first push to `next`:

1. Create the `next` GitHub environment with the same variables as `stage`: `GAR_BASE`, `GCP_ARTIFACTS_SA`, `GCP_DEPLOYMENTS_SA`, `GCP_PROJECT`, `GCP_REGION`, `GCP_SUBNETWORK`, `GCP_WIF`, and `GCP_ZONE`. The deploy jobs read their Google Cloud identity from it.
2. Reserve the six external addresses in `zfnd-dev-zebra`, before the first deploy. The workflow binds an address only when it creates a MIG, so an address reserved later has to be bound by hand.

   ```bash
   for NET in mainnet testnet; do
     for SUFFIX in "" -secondary -tertiary; do
       gcloud compute addresses create "zebra-next-${NET}${SUFFIX}" \
         --project zfnd-dev-zebra --region us-east1
     done
   done
   ```

3. Add a branch ruleset for `refs/heads/next` that requires a pull request with no approvals, requires the `lint`, `unit-tests`, `test-crates`, `pr-gate-result`, and `merge-policy` checks, allows only merge commits, and blocks force pushes and deletion.
4. Create `next` from `main`:

   ```bash
   git push origin origin/main:refs/heads/next
   ```

5. Deploy the six nodes. A new branch that matches `main` changes no files, so its first push may not pass the deploy workflow's path filter.

   ```bash
   for NET in Mainnet Testnet; do
     for ZONE in us-east1-b us-east1-c us-east1-d; do
       gh workflow run zfnd-deploy-nodes-gcp.yml -R ZcashFoundation/zebra --ref next \
         -f network="$NET" -f zone="$ZONE" -f environment=next \
         -f need_cached_disk=true -f cached_disk_type=tip
     done
   done
   ```

From then on, every merge into `next` that changes code deploys all six nodes.
