# GitLab Onboarding Pipeline

A GitLab CI pipeline that onboards and offboards users by reading two data files and making GitLab match them through the REST API (`/api/v4`). Each change to a data file is a request; the pipeline works out what changed and applies it.

## Ground rules

- **Variables.** Anything that can differ between environments comes from an environment variable: domain, URLs, naming formats, file paths, the default repo list, access levels and token settings. Non-secret defaults go in the `variables:` block of `.gitlab-ci.yml`. Secrets go in the project's CI/CD settings as masked, protected variables. Code reads them through one config module, and every value comes from there. See [Variables](#variables).
- **Ensure.** Every resource action is an `ensure_*` operation. First GET the resource. If it doesn't exist, create it. If it exists with the wrong settings, update it. If it exists and is correct, skip it. Running the same pipeline twice must give the same end state, and the second run should log only skips.
- **Log every decision.** Each `ensure_*` call writes exactly one line in the form `[ACTION] <resource-type> <name>: <reason>`. ACTION is one of `SKIP`, `CREATE`, `UPDATE`, `BAN`, `UNBAN`, `DELETE`, `RESTORE`, `COMMIT` or `ERROR`. Example: `[SKIP] user alice_RW: exists with expected config`. Mask tokens in all output.
- **Token scope.** Admin-level actions use `ADMIN_PAT`. Group and repository actions use the RW user's temporary PAT. The table in [Workflow](#workflow) says which token each action uses.
- **Modules.** Each resource type lives in its own module. See [Modules](#modules).

## Modules

There is one module per resource type. Each module exposes only `ensure_*` / `ban_*` / `delete_*` functions for its resource. The caller passes in an API client already bound to a token. Modules never pick a token, never read the data files and never call each other. The orchestrator is the only place that knows the phase order and which token goes to which call.

| Module | Responsibility |
|---|---|
| `config` | Reads and validates every variable in [Variables](#variables) |
| `log` | Writes the `[ACTION] <resource-type> <name>: <reason>` line |
| `api` | Thin GitLab client: base URL, auth header, pagination, error raising |
| `diff` | Finds the base SHA, loads both data files at base and head, and returns the added and removed sets |
| `validate` | Phase 1 checks |
| `git_ops` | Commit and push, reset-and-redo on a rejected push |
| `users` | Ensure user (create, update, unban), ban user |
| `tokens` | Create, revoke and sweep the temp PAT |
| `groups` | Ensure group |
| `members` | Ensure group membership at an access level |
| `projects` | Ensure project (create or restore), mark project for deletion |
| `main` | Orchestrator: runs the phases and wires tokens to modules |

Each resource module can be tested on its own against a mocked `api` client.

## Data files

Both are YAML maps keyed by username. Their paths come from `USERS_FILE` and `REPOS_FILE`.

This pipeline doesn't use the public keys. They're there for a separate pipeline, which uses each user's key to check who pushed a tarball before committing and pushing it to that user's repositories. This pipeline only checks that each new username has a key. It never uploads a key to GitLab or attaches one to the RW or RO user.

```yaml
# users.yaml: username -> public key
alice: "ssh-ed25519 AAAA... alice@laptop"

# repositories.yaml: username -> list of repositories
alice: [api, frontend]
```

## Trigger and concurrency

- The pipeline runs on pushes to the default branch that change `USERS_FILE` or `REPOS_FILE` (`workflow:rules` + `changes`).
- Pipelines run one at a time, in FIFO order. The job sets `resource_group: onboarding`. Gotcha: a resource group's default `process_mode` is `unordered`, and it can only be changed through the API, not in YAML. During setup, run `PUT /projects/:id/resource_groups/onboarding` with `process_mode=oldest_first` once.
- Commits the pipeline pushes itself are pushed with `-o ci.skip` so they don't trigger another pipeline. The exception is the validation fix in phase 1, which is meant to trigger one.

## Computing the diff

Compare pushes, not commits. The **base** is the SHA of the last pipeline that succeeded on this branch. The **head** is `CI_COMMIT_SHA`. For example, if the last successful pipeline ran on commit 3 and this push ends at commit 7, diff 3 → 7. Commits 4–6 are handled together no matter how they were pushed.

- To find the base, call `GET /projects/:id/pipelines?ref=$CI_COMMIT_BRANCH&status=success&order_by=id&sort=desc` with `ADMIN_PAT`, and take the `sha` of the first pipeline whose `id` isn't `CI_PIPELINE_ID`. Log the base and head SHAs.
- A failed pipeline never becomes the base, so the next push re-applies its changes. This is safe because every step is an ensure.- If no successful pipeline exists yet (first run), diff against an empty base, so every entry counts as added.
- Fetch full history (`GIT_DEPTH: 0`) so the base commit is available.

Diff the parsed data, not the text. Load each file at both SHAs with `git show <sha>:<path>`, treating a missing file as `{}`, and compare the two maps. A text diff breaks on reordering, quoting and multi-line values.

- `USERS_FILE`: keys **added** or **removed**. A changed key value needs no action here.
- `REPOS_FILE`: keys added or removed, plus repos added or removed within each key's list.

## Workflow

Run these phases in order. Phase 1 must finish before any API write.

**1. Validate.** Two checks, both against `CI_COMMIT_SHA`:

| Offending entry | Counts as fixed at the latest commit when |
|---|---|
| Username added to `USERS_FILE` with an empty public key | it has a public key, or has been removed |
| Username added to `REPOS_FILE` that isn't a key in `USERS_FILE` | it is now in `USERS_FILE`, or has been removed from `REPOS_FILE` |

If any check fails, make no API changes, not even for the valid entries in the same push. Then remove each offending entry:

1. Fetch the branch and hard-reset to `origin/$CI_COMMIT_BRANCH`.
2. Re-check each offending entry at that tip:
   - If it is fixed, log `[SKIP]` with the reason "fixed at <tip sha>". The push that fixed it has its own queued pipeline.
   - Otherwise remove that entry from its file and nothing else, and log `[ERROR]` with the reason, e.g. "no public key, removed from USERS_FILE" or "not in USERS_FILE, removed from REPOS_FILE".
3. If any entry was removed, commit and push **without** `ci.skip`. The new pipeline then onboards the valid usernames from the cleaned file. This is the one pipeline-made push that is meant to trigger a run. If the push is rejected, go back to step 1. Give up with `[ERROR]` after `PUSH_MAX_ATTEMPTS` tries.
4. Exit non-zero. This pipeline must never succeed. If it did, it would become the next run's diff base. An offending username that is later fixed would then look like a changed value instead of an added username, and it would never be onboarded.

**2. Onboard each added username `<u>`.**

| Step | Token |
|---|---|
| Ensure user `RW` login: `can_create_group: true`, `skip_confirmation`, `force_random_password`, email `<login>@EMAIL_DOMAIN` | ADMIN_PAT |
| Ensure user `RO` login: `can_create_group: false`, `external: true`, `skip_confirmation`, `force_random_password`, email `<login>@EMAIL_DOMAIN` | ADMIN_PAT |
| Create the temp PAT for the RW user | ADMIN_PAT |
| Ensure group `<u>` (path and name) | RW PAT |
| Ensure RO user is a member of the group at `RO_ACCESS_LEVEL` (20, Reporter) | RW PAT |

A user who already exists but is banned is a wrong config, so `ensure` unbans them (`[UNBAN]`).

**3. Offboard each removed username `<u>`.** Ban both RW and RO with `ADMIN_PAT` (`POST /users/:id/ban`; skip if already banned). Then clean up `REPOS_FILE`. Others may have pushed new users, repos or removals since the triggering push, so always work from the latest branch tip, never from `CI_COMMIT_SHA`:

1. Fetch the branch and hard-reset to `origin/$CI_COMMIT_BRANCH`.
2. If `<u>` is back in `USERS_FILE` at that tip, log `[SKIP]` and stop. Its own queued pipeline will re-onboard it.
3. If `<u>` is not a key in `REPOS_FILE` at that tip, log `[SKIP]` and stop.
4. Otherwise remove the `<u>` key and nothing else, commit, and push with `ci.skip`.
5. If the push is rejected, go back to step 1 and repeat the whole sequence on the new tip. Never rebase or merge this commit. Give up with `[ERROR]` after `PUSH_MAX_ATTEMPTS` tries.

This pipeline makes no API changes for the newer commits. Each push that changes a data file triggers its own pipeline, which runs next in the FIFO queue.

**4. Repositories, using the user's RW PAT.**
- Key added, or repos added to a list: ensure each repo in (added repos ∪ `DEFAULT_REPOS`) exists as a project in group `<u>`.
- Repos removed from a list: mark those projects for deletion, unless the repo is in `DEFAULT_REPOS`. A default repo is kept, logged as `[SKIP]` with the reason "default repo".
- A repo that is in both a user's list and `DEFAULT_REPOS` is valid. Ensure it once.
- Key removed entirely: check whether `<u>` is a key in `USERS_FILE` **at `CI_COMMIT_SHA`**. Check the file itself, not this diff, because the user may have been removed in an earlier push. If `<u>` is absent, log `[SKIP]` and continue (phase 3 handled it). Otherwise mark every project in group `<u>` for deletion. The group itself stays.

**Deletion is always soft.** "Delete" means one `DELETE /projects/:id`, which marks the project for deletion so that it can still be restored after a mistake (`POST /projects/:id/restore`). The pipeline never sends `permanently_remove`. The 6-month retention period is a GitLab instance setting, not something the pipeline controls. A project that already has `marked_for_deletion_at` set gets `[SKIP]`, with the reason "already marked for deletion".

**Re-added repos are restored.** When ensuring a project, if it exists but has `marked_for_deletion_at` set, restore it with `POST /projects/:id/restore` (RW PAT) and log `[RESTORE]`. Never create a new project next to it, because restoring keeps the old history. Gotcha: some GitLab versions rename a marked project's path, for example to `<path>-deletion_scheduled-<id>`. So a lookup by path can miss it. To find it, list the group's projects including those marked for deletion, and match on the original name. Check this behaviour on the target GitLab version.

## Temporary RW PAT

- Create it with `POST /users/:id/personal_access_tokens` using `ADMIN_PAT`. Use scopes from `RW_PAT_SCOPES`, `expires_at` = today + `RW_PAT_EXPIRY_DAYS`, and a name starting with `onboarding-ci-` and ending in `CI_PIPELINE_ID`.
- Revoke it on every exit path, success or failure, with `DELETE /personal_access_tokens/:id`. Use `try/finally` plus `after_script`, which also covers a job that times out. The short expiry is a backstop for jobs that get killed.
- At startup, revoke any leftover `onboarding-ci-*` tokens from earlier runs that died, and log each one as `[DELETE]`.
- Create one PAT per RW user, only when that user has group or repo work in this run. That includes existing users who only appear in `REPOS_FILE`.

## Variables

| Variable | Purpose | Default |
|---|---|---|
| `GITLAB_URL` | GitLab instance base URL | `$CI_SERVER_URL` |
| `EMAIL_DOMAIN` | Domain for user emails | (none, required) |
| `USERNAME_TEMPLATE` | Login name format | `{username}_{role}` |
| `RW_ROLE` / `RO_ROLE` | Role values used in the template | `RW` / `RO` |
| `USERS_FILE` / `REPOS_FILE` | Data file paths | `users.yaml` / `repositories.yaml` |
| `DEFAULT_REPOS` | Comma-separated repos created for every user | (empty) |
| `RO_ACCESS_LEVEL` | RO user's group access level | `20` |
| `RW_PAT_SCOPES` | Temp PAT scopes | `api` |
| `RW_PAT_EXPIRY_DAYS` | Temp PAT lifetime (backstop) | `1` |
| `ADMIN_PAT` | Admin token (masked, protected) | (none, required) |
| `PUSH_MAX_ATTEMPTS` | Tries before giving up on a pipeline-made push (validation fix or cleanup) | `5` |
| `PUSH_TOKEN` | Token used to push validation fixes and cleanups (`write_repository`) | (none, required) |

When you add a variable, add it to this table, to `.gitlab-ci.yml` (if it isn't a secret) and to the config module.
