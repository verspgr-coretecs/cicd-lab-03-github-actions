# Lab 03 — GitHub Actions

**A hands-on workshop.** One continuous arc: take the Ignition project you already
know and build a **CI safety net** around it — local linters first, then a GitHub
Actions workflow that makes those checks a required gate on every PR, then a look at
*where* CI runs when GitHub-hosted isn't enough.

> The subject is the same Ignition gateway and Perspective project from Lab 02 (a
> Oatmakers Site 04 oat-line "Overview" HMI plus two Jython script libraries). You're not learning
> new application code — you're adding CI around code you already understand. The repo
> tracks only the project files under `projects/`; the gateway generates its own config
> into a Docker volume we never commit. (Lab 04 goes deep on that file layout.)

**What this lab is — and isn't.** Today you build the *safety net*: CI that catches problems
in your project files before they ship. You won't deploy anything to a gateway yet —
automated, file-based deployment is the subject of Labs 04–05. The goal here is to make
"broken project files can't reach `main`" automatic.

## What you'll do

- **Part 1 — Linters as your safety net:** yamllint, shellcheck, actionlint, **ign-lint**, `scripts/validate.sh`
- **Part 2 — GitHub Actions:** build `ci.yml`, path filters, required check
- **Part 3 — Self-hosted runners:** a look ahead (short demo) — hands-on comes in Labs 04–05

## Setup

You'll need Docker (Compose V2) and Python 3.10+. Clone, copy the env file, and confirm
the gateway boots:

```bash
cp .env.example .env
scripts/setup.sh        # boots one Ignition gateway, waits for RUNNING, prints the URL + login
# open http://localhost:8088  → log in with the .env credentials
```

> **"Port is already allocated"?** That's your **Lab 02 gateway**, still running from
> yesterday — its `restart: unless-stopped` policy keeps it alive, reboots included. This
> collision is left in deliberately: a port is a machine-wide resource, and two gateways
> can't share one. Tear Lab 02 down first (its state survives in its Docker volume):
> `cd ../cicd-lab-02-branching-and-prs && scripts/teardown.sh`, then re-run
> `scripts/setup.sh` here.

Install the linters:

```bash
# macOS — Apple's bare `python3` is 3.9, but ign-lint needs 3.10+,
# so install a modern Python (3.12) alongside the linters:
brew install shellcheck actionlint python@3.12

# Debian/Ubuntu/WSL — shellcheck via apt; actionlint has NO apt package,
# so grab the release binary and put it on your PATH:
sudo apt install shellcheck python3-venv
bash <(curl -sSL https://raw.githubusercontent.com/rhysd/actionlint/main/scripts/download-actionlint.bash)
sudo mv actionlint /usr/local/bin/

# both platforms — venv on Python 3.10+ (macOS: python3.12 · Linux: python3),
# which also keeps the pinned tools out of your system Python:
python3.12 -m venv .venv && source .venv/bin/activate   # Linux: python3 (already 3.10+)
pip install yamllint==1.35.1 ign-lint==0.6.1            # ign-lint needs Python 3.10+
```

> **Build the venv with Python 3.10+.** `ign-lint` publishes wheels only for Python
> `>=3.10,<3.14`, so a venv on an older interpreter fails at `pip install` with
> `No matching distribution found for ign-lint` (pip filters it out by `requires-python`,
> reporting `from versions: none`). macOS's system `python3` is **3.9**, which triggers
> exactly this — name `python3.12` explicitly. On Debian/Ubuntu/WSL the system `python3`
> is already 3.10+, so plain `python3` is fine. Confirm with `python -V` after activating.
> CI runs the same pins on Python 3.12.

> **Why the venv?** A bare `pip install` on Homebrew or Ubuntu 24.04+ Python fails with
> `error: externally-managed-environment` (PEP 668); the venv sidesteps that and keeps
> the lab's pinned tool versions out of your system Python. `.venv/` is already
> gitignored. Re-activate in new terminals with `source .venv/bin/activate`. On a
> minimal Debian/Ubuntu box, install the venv module first: `sudo apt install
> python3-venv`. (Skip `brew install yamllint` — the pinned pip version is the one CI uses.)

> **About `ign-lint`:** it's the one Ignition-specific tool here, a young pre-1.0 linter
> (v0.6.1) from [BW Design Group](https://github.com/bw-design-group)
> ([ignition-lint repo](https://github.com/bw-design-group/ignition-lint)). Worth knowing the
> history: the project started as [`etknorr/ignition-lint`](https://github.com/etknorr/ignition-lint),
> which is now **archived read-only** (June 2026); active development moved to the BW Design Group
> fork we pin here, and it carries a richer rule set. That handoff is itself the lesson: Ignition-native
> CI tooling is still young, so you pin a version, watch where the maintained line lives, and treat
> the tool as a great example of the *pattern* (an Ignition-aware linter as a required CI check)
> rather than a permanent dependency.

### Your own repo (needed for Part 2)

Part 2 has you open pull requests and set a *required status check* on `main` — both need
a repo **you** control (you need admin rights to configure branch protection). You already
forked the lab and cloned your fork, so `origin` is your own repo — nothing more to set up.
Just point `gh` at your fork so PRs land there and not on the source repo:

```bash
# make gh target YOUR fork for PRs, not the source repo:
gh repo set-default <you>/cicd-lab-03-github-actions
```

Confirm `origin` is your fork with `git remote -v`. (No `upstream` remote is needed — the
lab never pulls from the source repo.)

Two fork gotchas, both one-time:

1. **Enable workflows on the fork.** GitHub keeps Actions dormant on fresh forks: open your
   fork's **Actions** tab on github.com and click *"I understand my workflows, go ahead and
   enable them"*. Until you do, PRs in your fork run **no CI at all**.
2. **Watch the PR base.** New pull requests default to the *source* repo
   (`Mustry-Academy/...`). The `gh repo set-default` above fixes the CLI; in the web UI,
   check that the base branch is **your fork's** `main` before you click create.

Open PRs inside your own fork: you own it, so you can configure branch protection and merge
your own PRs once CI is green.

Reference reading lives in [`docs/validation-and-linters.md`](../docs/validation-and-linters.md)
and [`docs/self-hosted-runners.md`](../docs/self-hosted-runners.md).

---

## Part 1 — Linters as your safety net

**Goal:** run yamllint, shellcheck, actionlint, **ign-lint**, and `scripts/validate.sh`
against a real Ignition project; read each tool's output; decide whether to fix,
configure away, or ignore each finding; and tune `.yamllint.yml` to fit the project.

Every linter is a cheap, fast check that catches one class of bug. The point isn't to
run them all — it's to know which one would have caught yesterday's regression.

### Seed the broken state

```bash
scripts/seed.sh
```

This plants a handful of issues into your working tree — at least one for every tool,
including three realistic Ignition findings in the Perspective view: a **brittle, broken
binding**, a **runaway poll rate**, and a **mis-named component**. Hunt them down with the
linters — and resist `git diff`, which spoils every answer at once; reading the linters'
output is the exercise. Reset to a clean tree
any time with:

```bash
git restore . && rm -f .github/workflows/example.yml
```

### We-do

The instructor live-demos on the seeded state. For each tool: run it, read the output,
fix one finding.

> **These are separate commands, not one runner.** There is no single "lint everything"
> button in this lab. `scripts/validate.sh` is its own small script (item 2); the four
> linters below it are each their own tool you install and run separately. `validate.sh`
> is deliberately the *cheapest, broadest* check (does every file parse?), so it runs even
> with none of the linters installed. The linters go *deeper* on narrower things.

1. `yamllint -c .yamllint.yml docker-compose.yml`: YAML syntax + style (finds trailing whitespace).
2. `scripts/validate.sh`: the gateway-free green/red signal, and **not a linter**. It only checks
   that every `*.json` under `projects/` is valid JSON and every `code.py` parses as Python 3.
   Exit 0 = green, 1 = red. The same check the PR uses. It does **not** run yamllint/shellcheck/
   actionlint/ign-lint for you; those are the separate commands in this list. (Honest fine
   print: the gateway runs **Jython 2.7**; this is a fast Python-3 *parse* check, not a Jython
   validator. Write Python-3-parseable syntax and you're fine — the lab's scripts already are.)
3. `ign-lint --config rule_config.json --files "projects/**/view.json"`: **the flagship
   tool of this part.** Ignition-native static analysis: it parses the Perspective
   `view.json`, walks the component tree, and checks naming conventions, binding poll
   rates, brittle references, and the Python embedded in views, all without a running
   gateway.
4. `actionlint`: GitHub Actions workflow syntax + expression typing (run it on the seeded `example.yml`).
5. `shellcheck scripts/*.sh`: catches almost every shell scripting bug ever made.

Spend the most time on **ign-lint** — it's the one that's genuinely Ignition-aware, and
the one most people here have never seen. Open `rule_config.json` and walk the rules:
`NamePatternRule` (components → PascalCase, properties → camelCase, message handlers →
kebab-case, custom methods → snake_case), `PollingIntervalRule` (a floor on binding poll
rates — 1000 ms here), `BadComponentReferenceRule` (flags `.getSibling()` / `.getParent()`
traversal), `PylintScriptRule`, and the rest. The clean `lab-project` passes ign-lint with
**zero** findings — every finding you see is something `scripts/seed.sh` broke on purpose.

### You do

Hunt every planted issue, fix them all, then make the config your own. (The step numbers
match the assignment slides.)

1. Run `scripts/seed.sh` for a fresh broken state (reset first if your tree still carries
   We-do fixes: `git restore . && rm -f .github/workflows/example.yml`).
2. Run each of `yamllint`, `shellcheck`, `actionlint`, `ign-lint`, and `scripts/validate.sh`,
   and write down each finding — make a list: *what the tool flagged*, in *which file*.
3. Spend the most time reading **ign-lint**'s output. It fires two rules on one binding
   (the Kiln tile), plus a poll-rate finding and a naming finding. Work out *why*
   each rule exists before fixing anything.
4. Fix every finding. For each, record: *what the tool flagged*, *why your fix is correct*,
   and *what class of production bug it would catch*.
5. Re-run every linter until each is silent and `scripts/validate.sh` exits 0.
6. Open `.yamllint.yml`. We disabled `line-length` for the project — **extend the comment**
   explaining *why* (hint: long compose environment lines).
7. Commit. Your end state should be a clean tree: every linter silent, `scripts/validate.sh`
   exits 0.

> **No delete-cheats.** Don't silence ign-lint by deleting the Clock, the Kiln binding,
> or the Power tile — only the *reference*, the *poll rate*, and the *name* were broken, so
> restore real data bindings and a real name. The view must still load, and you can prove it
> in the gateway: `docker compose restart` (or `scripts/scan.sh` with an API key in `.env`),
> then reload the Overview page. A running gateway only picks up bind-mounted edits on a
> project scan or a restart.

> Stuck on a finding? [`instructor-notes/lab-key.md`](../instructor-notes/lab-key.md) has the
> walkthrough — but give it a genuine attempt yourself first; the diagnostic skill is most
> of the lesson.

### Stretch `[OPTIONAL]` — block bad commits before they leave your machine

CI catches it on the PR; [`pre-commit`](https://pre-commit.com) catches it at `git commit`.
The repo ships the config: `.pre-commit-config.yaml` wires all four linters.

1. Install and enable the hook: `pip install pre-commit`, then `pre-commit install`.
2. Baseline run over everything: `pre-commit run --all-files` — should be clean after Part 1.
3. **Prove it works:** make a bad change (set a binding to `now(250)`), try to commit, and
   watch the ign-lint hook refuse the commit.
4. Undo the bad change. If the commit went through anyway: did `pre-commit install` run? Is
   `.git/hooks/pre-commit` populated?

> **Two nets.** Pre-commit is the *fast, local* net; CI is the *enforced, shared* one. You
> want both: local for speed, CI because hooks are opt-in and can be skipped.

### Debrief

- Which of these linters would have caught the most recent real bug your team shipped?
- When does linting *hurt* instead of help? (When it flags style as errors; when it's
  slower than the dev loop; when its config drifts from reality.)
- ign-lint is new to most of you: which of its rules map to a mistake you've actually
  shipped in a Perspective project? Which feel like overreach?

---

## Part 2 — GitHub Actions: workflows, jobs, required checks

**Goal:** write a workflow from scratch with sensible defaults (PR trigger,
least-privilege permissions, path filters), wire in the Part 1 linters, and make the
whole thing a *required check* no one can merge past.

Start from a clean tree (Part 1 fixes applied, `.yamllint.yml` in place).

> **The one idea in this Part: a validator is already a pipeline.** In Part 1 you ran
> `scripts/validate.sh` and the linters *by hand* at a terminal. That is a pipeline: an event
> (you typing) triggers steps (the commands) that pass or fail (the exit code). All Part 2 does
> is change the trigger from "you remembering" to "a pull request", and move the steps into
> `ci.yml`. Same steps, same pass/fail, different trigger. When you build the workflow below,
> notice you're not writing new *checks*, you're just wrapping the ones you already ran. (And
> this afternoon in Lab 04, a *deploy* is the same shape again: trigger, steps, pass/fail, where
> the steps ship files to a gateway.)

```
workflow  ──contains──▶  jobs  ──contains──▶  steps  ──run──▶  actions or shell commands
   │                       │                      │
   │                       │                      └── env, working-directory…
   │                       └── runs-on, needs, strategy, permissions
   └── on (triggers), permissions, concurrency
```

The repo ships the **finished** workflow — it's the answer key, and your fork has it too.
Set it aside first, so you're building your own rather than admiring ours (you'll compare
against it in step 4).

The quickest path is `scripts/prepare-part2.sh` — it prints each action as it runs, does the
`git mv`, and scaffolds a bare two-job skeleton (`validate` + `lint`, each just `checkout` plus
TODO markers) into a fresh `.github/workflows/ci.yml`:

```bash
scripts/prepare-part2.sh
git add -A && git commit -m "chore: set aside reference workflow, scaffold ci.yml skeleton"
```

Prefer to do it by hand? The script only runs the `git mv` and writes that skeleton file — so
just do both yourself:

```bash
git mv .github/workflows/ci.yml .github/ci-reference.yml
git commit -m "chore: set aside the reference workflow"
```

Either way, `.github/workflows/ci.yml` is now yours to build out — filling in the TODOs, not
copying the reference. Start with the gateway-free validator job:

```yaml
name: CI
on:
  pull_request:
permissions:
  contents: read
jobs:
  validate:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4
      - run: scripts/validate.sh
```

Open a PR, watch it run, read the logs together — each step is its own collapsible block.
Then flesh out the `lint` job with the Part 1 linters, including `ign-lint`:

```yaml
  lint:
    runs-on: ubuntu-latest
    timeout-minutes: 10   # this job installs tools; give it more headroom than validate
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install yamllint==1.35.1 ign-lint==0.6.1
      - run: yamllint -c .yamllint.yml .
      - uses: raven-actions/actionlint@v2
      - run: sudo apt-get update && sudo apt-get install -y --no-install-recommends shellcheck
      - run: shellcheck scripts/*.sh
      - run: ign-lint --config rule_config.json --files "projects/**/view.json"
```

`ign-lint` needs Python 3.10+, which is why we pin `setup-python` to `"3.12"`. Discuss as
you go:

- **`permissions: contents: read`** — least privilege; the default `GITHUB_TOKEN` is too broad.
- **`GITHUB_TOKEN`** — auto-provisioned per job, scoped to the repo, expires when the job ends.
- **Secrets vs variables** — secrets are encrypted and masked (`***`) in logs; variables
  are plain text. Live-add an `EXAMPLE_SECRET` and confirm it's masked.
- **Job ids are check names** — branch protection (You-do step 3) matches status checks
  by name, and a job's name *is* its id (`lint`, `validate`) unless you override it with
  `name:`. Rename a job later and any required check pointing at the old name waits forever.
- **`timeout-minutes`** — a hung job otherwise runs (and bills) for up to 6 hours;
  capping every job is free insurance. Size it per job: a required check that flakes
  because a slow apt mirror blew a too-tight cap blocks the whole team.

### You do

Make the workflow yours.

**1 — Path filters.** Add a `paths:` filter so the workflow skips docs-only PRs:

```yaml
on:
  pull_request:
    paths:
      - "projects/**"
      - "scripts/**"
      - "docker-compose.yml"
      - ".github/workflows/**"
      - ".yamllint.yml"
      - "rule_config.json"
  push:
    branches: [main]
```

Open a PR that touches **only** `README.md` and confirm the workflow is **skipped** (not
just passed). Hold that thought — it collides with required checks in step 3.

**2 — Compose validation.** Add a final step to the lint job:

```yaml
      - run: docker compose config -q
```

This catches Compose-level issues yamllint can't see — undefined services, port-string
typos, malformed environment maps.

**3 — Protect `main`.** In repo settings, add a branch protection rule for `main` with four
things enabled — and grant yourself no bypass, or the rules won't apply to you as the
repo admin:

- **Require a pull request before merging** — blocks direct commits
- **Require status checks to pass** — select **`lint`** and **`validate`**
- **Block force pushes** — no rewriting history
- **Restrict deletions** — `main` can't be deleted

(If your Settings page offers the newer **rulesets** UI instead, the same four rules exist
there — but leave **Restrict updates** *unticked*: it doesn't just block direct pushes, it
blocks PR merges too.)

Now prove the wall exists, from both sides:

1. **The forbidden route.** Commit straight to `main`
   (`git commit --allow-empty -m "test: direct to main"`) and push. GitHub rejects the
   push — direct commits to `main` are dead.
2. **The proper route.** Make a change on a branch, open a PR, wait for `lint` and
   `validate` to go green, and merge. The merge button is now the only door in.
3. **Break it on purpose.** Run `scripts/seed.sh` to plant the broken state (or just set the
   Clock's poll to `now(250)` by hand), commit on a branch, and open a PR. Confirm GitHub
   blocks the merge. Then fix and re-push — but fix *forward* (e.g. `now(1500)`), don't
   revert to the exact original value: a PR whose net diff against `main` is empty matches
   no `paths:` filter, so CI never re-reports and the required checks sit on "Expected"
   forever — the step-1 trap in a second costume. (If you did revert exactly, push any real
   change, or close the now-empty PR; there's nothing left to merge anyway.)

> **The trap you just set.** Required checks and `paths:` filters interact badly. Your
> docs-only PR from step 1 was *skipped* — but a required check that never reports
> doesn't pass, it stays **"Expected — waiting for status" forever**. Try it: open
> another README-only PR now and watch it hang. With branch protection on, nobody can
> merge a docs-only change without an admin override. GitHub's documented fix is a
> **twin no-op workflow** with the **same job names** — the job name is what a required
> check matches on — and an inverse `paths-ignore:` filter, so it reports an instant
> green `lint` and `validate` on exactly the PRs the real CI skips
> ([Handling skipped but required checks](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/defining-the-mergeability-of-pull-requests/troubleshooting-required-status-checks#handling-skipped-but-required-checks)).
> Building it is a stretch goal below; for this lab repo it's also fine to just
> understand *why* the PR hangs — this exact interaction bites real teams.

**4 — Sanity check.** Commit any remaining changes. Your workflow should *structurally*
match the reference you set aside at the start of this Part (`.github/ci-reference.yml`) —
same triggers, jobs, and step order; step `name:` labels and comments may differ — see
[`instructor-notes/lab-key.md`](../instructor-notes/lab-key.md) for the walkthrough.

### Stretch `[OPTIONAL]`

- **Fix the docs-only-PR hang** with the no-op twin from the step 3 callout: a second
  workflow whose `paths-ignore:` mirrors the real filter list, with jobs named exactly
  `lint` and `validate` that just `echo` and exit 0 — the *job* names are what the
  required checks match on (the workflow's `name:` is cosmetic). Mind the caveat in
  GitHub's docs: a PR touching both docs *and* code triggers both workflows, and two
  check runs then report under each name.
- **Add the manual Run button.** From the triggers slide: add `workflow_dispatch:` to your
  `on:` block and push. The Actions tab grows a *Run workflow* button — click it and CI runs
  without a PR or a push. (Part 3's runner demo uses the same trigger.)
- **Watch GitHub censor a secret, in your own fork.** You saw the demo in the We-do; now do
  it yourself: `gh secret set EXAMPLE_SECRET --body "hunter2"`, add a step that runs
  `echo "leak? ${{ secrets.EXAMPLE_SECRET }}"`, push, and read the log — it prints
  `leak? ***`. Delete the echo step afterwards; the exercise is watching the masking work,
  not keeping a leak around.
- **Cancel superseded runs.** Add a workflow-level `concurrency:` group
  (`group: ${{ github.workflow }}-${{ github.ref }}`, `cancel-in-progress: true`) so a
  force-push doesn't leave a stale run burning minutes — the `concurrency` box from the
  mental-model diagram, in practice. Prove it: push two commits in quick succession and
  watch the first run flip to *cancelled*.

### Debrief

- Where does the workflow actually run? (An ephemeral runner spun up per job.)
- What happens when a step fails midway? A whole job? (`continue-on-error`, `needs:`.)
- Required checks are a contract with the *team*, not just a setting — who decides what's blocking?
- We pin pip packages to an exact version but actions to a mutable tag (`@v4`). What's the
  difference in risk? (A tag can be moved by the action's owner; supply-chain-sensitive repos
  pin actions to a full commit SHA and let Dependabot bump them. This matters more once a
  self-hosted runner sits inside a plant network — Part 3.)

---

## Part 3 — Self-hosted runners (a look ahead + optional hands-on)

The instructor frames self-hosted runners with the class: what they are, when you'd reach for
one, and the one rule that matters. You'll get **hands-on for real** in Labs 04–05 — deploying
to a real gateway usually means a runner that can reach it. An **optional** hands-on at the end
of this Part lets you try the whole loop now, on your throwaway fork.

**When GitHub-hosted is enough — most of the time.** Your code lints and validates on a
standard image and you're not blocked by network or compliance. Use GitHub-hosted and move on.

**When you need self-hosted.** The big one for Ignition is **network isolation**: the gateway
you deploy to sits behind a firewall (on-prem, a customer VPN, a private PLC network) that a
GitHub-hosted runner can't reach. (Also: compliance / data residency, real hardware, and cost
at very high volume.)

**The one rule to remember.** A self-hosted runner executes arbitrary workflow code on a
machine inside your network. *Never* attach one to a **public** repo that accepts **fork PRs**
— a malicious PR can run code on your network. Use *ephemeral* runners and trusted repos.

**Discussion (everyone).** Which of *your own* deploys would need a self-hosted runner — and
why, specifically? ("Our gateway is behind the customer's VPN; GitHub-hosted physically can't
reach it" beats "maybe.")

### Optional hands-on — try the loop yourself `[OPTIONAL]`

> **Go / no-go — read first.** This is safe *only* because the runner is **ephemeral**, the
> demo workflow is **`workflow_dispatch`** (so no fork PR can trigger it), it runs on your
> **throwaway fork**, and you **tear it down** at the end. Do **not** reproduce this on a repo
> you care about — that is exactly the fork-PR footgun the rule above warns against.

**1 — Register an ephemeral runner.** Mint a short-lived registration token (it uses your
existing `gh` auth and expires on its own in ~1 hour), then start the runner container. Watch
it appear in *Settings → Actions → Runners* and print `Listening for Jobs`:

```bash
# your repo — auto-detected, nothing to substitute
REPO=$(gh repo view --json nameWithOwner --jq .nameWithOwner)

# short-lived registration token
RUNNER_TOKEN=$(gh api -X POST \
  "repos/$REPO/actions/runners/registration-token" --jq .token)

docker run -d --rm --name lab03-runner \
  -e REPO_URL="https://github.com/$REPO" \
  -e RUNNER_TOKEN="$RUNNER_TOKEN" \
  -e LABELS="self-hosted,local-lab03" \
  -e EPHEMERAL=true \
  myoung34/github-runner:latest

docker logs -f lab03-runner          # watch for "Listening for Jobs"
```

**2 — Dispatch the shipped workflow.** The repo already ships
[`.github/workflows/runner-demo.yml`](../.github/workflows/runner-demo.yml) —
manual-only (`workflow_dispatch`), with `runs-on: [self-hosted, local-lab03]`, so there is
nothing to write. Fire it, then flip to the runner's logs:

```bash
gh workflow run runner-demo.yml   # then watch the docker logs terminal
```

The job runs live in `docker logs`, on your machine, not GitHub's infra. It is deliberately
a single `echo`, so it **goes green** — the proof is the hostname in the log: your container's,
not a GitHub machine's. The labels did the routing. Keep in mind a bare runner only has what
*you* put on it: the moment a job needs shellcheck or Python, "works on `ubuntu-latest`" is an
assumption GitHub's image quietly satisfied and yours won't — Labs 04-05 install a real
toolchain for exactly that reason. After the one job, the ephemeral container exits and the
runner drops off the Runners tab.

**3 — Tear it down (mandatory if you ran it).** Cleanup *is* the exercise — a lingering runner
on a public repo is the standing version of the fork-PR risk. The workflow itself is
`workflow_dispatch`-only and harmless, so leave it; the **runner** is what you remove:

```bash
docker rm -f lab03-runner   # no-op if the ephemeral runner already exited
```

Then on github.com: confirm *Settings → Actions → Runners* is empty, and revoke any classic
**PAT** you minted (the `gh api` registration tokens above expire on their own). The full
how-to is in [`docs/self-hosted-runners.md`](../docs/self-hosted-runners.md), and you'll do it
for real in Lab 04.

---

## Wrap-up & take-home

You built a CI safety net for an Ignition project, end to end:

- **Local linters** — yamllint, shellcheck, actionlint, **ign-lint**, and `scripts/validate.sh`
  — each catching a class of bug before it ships.
- **A GitHub Actions workflow** that runs them on every PR, with least-privilege permissions,
  path filters.
- **A required check** that turns "please run the linters" into "you cannot merge until they pass."
- **An understanding of self-hosted runners** — when they're worth it, and the security
  weight they carry.

**Before Lab 04:** skim [`docs/self-hosted-runners.md`](../docs/self-hosted-runners.md) — you'll register a real runner there.

**What's next:** Lab 04 opens up the Ignition file structure itself — `project.json`, view
exports, and how to deploy project files to a gateway properly — building on the CI
foundation you just laid.
