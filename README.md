# Bug Bounty Automation Using GitHub Actions 🔥

Documentation only. This repository contains no active workflows and runs nothing. The example lives in `examples/recon.yml` and is meant to be copied into a private repository.

Your recon should not depend on a server you SSH into. GitHub Actions gives you scheduled Linux machines that start clean, run your tools, and disappear.

One workflow returns every subdomain, every live host, and status reports split into 2xx, 3xx, 4xx, and 5xx files.

For operators who already run subfinder and httpx. This covers the automation layer, not the tools.

## Table of Contents

* [Quick Start](#quick-start)
* [How GitHub Actions Works](#how-github-actions-works)
  * [The execution model](#the-execution-model)
  * [Why it fits recon](#why-it-fits-recon)
  * [What it cannot do](#what-it-cannot-do)
  * [A minimal workflow](#a-minimal-workflow)
  * [Fields that matter](#fields-that-matter)
  * [Reading a run](#reading-a-run)
  * [Limits](#limits)
* [Bug Bounty Patterns](#bug-bounty-patterns)
  * [1. Schedule](#1-schedule)
  * [2. Manual run with a scope guard](#2-manual-run-with-a-scope-guard)
  * [3. Matrix sharding](#3-matrix-sharding)
  * [4. State in a git branch](#4-state-in-a-git-branch)
  * [5. Diff](#5-diff)
  * [6. Notify](#6-notify)
  * [7. Findings as issues](#7-findings-as-issues)
  * [8. Artifacts or state](#8-artifacts-or-state)
  * [9. Cadence](#9-cadence)
  * [10. Tuning](#10-tuning)
* [OPSEC and Cost](#opsec-and-cost)
  * [Network identity](#network-identity)
  * [Attribution](#attribution)
  * [Scope](#scope)
  * [Terms](#terms)
  * [Data handling](#data-handling)
  * [Cost](#cost)
  * [When to move to a VPS](#when-to-move-to-a-vps)
  * [Checklist](#checklist)
* [Example: Subdomain Enum](#example-subdomain-enum)
  * [Goal](#goal)
  * [Flow](#flow)
  * [Walkthrough](#walkthrough)
  * [Output](#output)
  * [Run it](#run-it)
  * [Tuning](#tuning)
  * [Extending](#extending)
  * [Failure modes](#failure-modes)
  * [Scope](#scope-1)
* [Repository Map](#repository-map)

## Quick Start

This repository is a guide, not a running pipeline. To use the example, copy it into a private repository of your own.

```bash
mkdir -p ~/bb-recon/.github/workflows
cp examples/recon.yml ~/bb-recon/.github/workflows/recon.yml
cd ~/bb-recon
git init -b main
git add .
git commit -m "recon workflow"
gh repo create bb-recon --private --source=. --push

gh workflow run recon.yml
gh run watch
gh run download --dir ./reports
```

The artifact lands in `./reports/recon-<run_number>/`.

Example `results/status.txt`:

```text
200	https://www.dell.com	Dell
204	https://cdn.dell.com	-
301	http://dell.com	-
403	https://support.dell.com	Forbidden
500	https://api.dell.com	Error
```

## How GitHub Actions Works

### The execution model

A workflow is one YAML file in `.github/workflows/`. GitHub evaluates it when an event matches.

```
event -> workflow -> job -> step
```

* Event: `push`, `pull_request`, `schedule`, `workflow_dispatch`.
* Workflow: the file. Holds `on`, `jobs`, and shared defaults.
* Job: one unit of work on one fresh machine.
* Step: one shell command or one packaged action.

Jobs are isolated and run in parallel unless `needs` says otherwise. Steps inside a job share a filesystem and run in order.

The runner starts empty. Nothing from your repository exists until `actions/checkout`.

### Why it fits recon

* Disposable compute. Every job gets a clean VM, so there is no state to clean up and no drift between runs.
* Built in scheduler. Cron without a crontab and without a machine to keep alive.
* Fan out. A matrix runs the same job with different values, up to 256 jobs per run.
* Parallel by default. Four nuclei shards start at the same time.
* Secrets. API keys and webhooks are encrypted and masked in logs.
* Artifacts. Results survive the VM and can be downloaded later.
* Git as state. Commit yesterday's output to a branch and diff against today.

### What it cannot do

* No inbound network. You cannot host a listener, a callback server, or a C2.
* No process that outlives the job. The 6 hour cap ends everything.
* No persistent filesystem. State must go to a git branch or an artifact.
* Shared cloud IPs. Targets can block or rate limit GitHub ranges.
* No interactive shell. You debug by logging.

### A minimal workflow

```yaml
name: demo

on: workflow_dispatch

jobs:
  hello:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - run: echo "runner is $RUNNER_OS"
```

Push it, open the Actions tab, and click Run workflow.

From the terminal:

```bash
gh workflow run demo.yml
gh run watch
gh run view --log
```

### Fields that matter

* `on`: events and filters. This decides when the job runs.
* `permissions`: what `GITHUB_TOKEN` can do. Keep it minimal.
* `concurrency`: stop overlapping runs. Useful for scans that should not stack.
* `env`: plaintext configuration such as the target domain.
* `runs-on`: the runner image.
* `timeout-minutes`: fail fast instead of burning the 360 minute default.
* `strategy.matrix`: fan out one job into many.
* `steps`: the actual work.

### Reading a run

* Summary page: job graph, timings, annotations, artifact links.
* Job log: one expandable section per step.
* `gh run view <id> --log-failed`: only the failing step.
* `gh run rerun <id> --failed`: retry just the failed jobs.

### Limits

* 6 hours per job on GitHub hosted runners.
* 35 days per workflow run, including time spent waiting for approval.
* 256 jobs per matrix.
* 20 concurrent jobs on Free, 40 on Pro, 60 on Team, 500 on Enterprise.
* 10 GB of cache per repository, evicted after 7 days without use.
* 500 MB of artifact storage on Free, shared with Packages.
* 1,000 API requests per hour for `GITHUB_TOKEN`.
* Cron runs in UTC with a 5 minute minimum and can be delayed under load.
* Scheduled workflows stop after 60 days of repository inactivity.

## Bug Bounty Patterns

These are the patterns worth copying into a recon repo. Each one is small on its own. Combined, they replace most of what a recon VPS does.

### 1. Schedule

```yaml
on:
  schedule:
    - cron: '17 3 * * *'
  workflow_dispatch:
```

Use a minute that is not zero. The top of the hour is the busiest queue.

Cron is UTC. A workflow scheduled on the default branch reads the YAML from that branch.

### 2. Manual run with a scope guard

```yaml
on:
  workflow_dispatch:
    inputs:
      target:
        type: string
        required: true

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - env:
          TARGET: ${{ inputs.target }}
        run: |
          if ! grep -qx "$TARGET" scope.txt; then
            echo "::error::$TARGET is not in scope.txt"
            exit 1
          fi
          ./scan.sh "$TARGET"
```

Never interpolate `${{ inputs.target }}` directly into a `run:` block. Pass it through `env` and let the shell treat it as data.

### 3. Matrix sharding

```yaml
strategy:
  fail-fast: false
  matrix:
    shard: [0, 1, 2, 3]

env:
  SHARD: ${{ matrix.shard }}

steps:
  - run: |
      awk -v s="$SHARD" -v n=4 'NR % n == s' hosts.txt > shard.txt
      nuclei -l shard.txt -silent -o "out/shard-$SHARD.txt"
```

Four shards run at the same time. This is the main reason to use Actions instead of a single VPS: horizontal fan out with no orchestration code.

Keep each shard under the 6 hour job limit. If a shard can exceed it, reduce concurrency or split again.

### 4. State in a git branch

Caches are not a database. They are evicted and they cannot be diffed. Use a branch.

```bash
git clone --depth 1 --branch state \
  "https://x-access-token:${GITHUB_TOKEN}@github.com/${GITHUB_REPOSITORY}.git" state \
  || mkdir state
```

Write today's output into `state/`, commit, and push. The next run clones it and can diff.

Keep the branch small. Store host lists and summaries, not screenshots or raw response bodies.

### 5. Diff

```bash
comm -13 \
  <(sort -u state/subdomains.txt) \
  <(sort -u results/subdomains.txt) > results/new-subdomains.txt
```

`comm -13` prints lines that exist only in the second file. That is your new attack surface for the day.

Use the count to decide whether to notify:

```yaml
- id: diff
  run: echo "count=$(wc -l < results/new-subdomains.txt)" >> "$GITHUB_OUTPUT"
```

### 6. Notify

```yaml
- name: Notify
  env:
    DISCORD_WEBHOOK: ${{ secrets.DISCORD_WEBHOOK }}
  run: |
    if [ -z "$DISCORD_WEBHOOK" ]; then
      echo "no webhook configured"
      exit 0
    fi
    count=$(wc -l < results/new-subdomains.txt)
    if [ "$count" -eq 0 ]; then
      exit 0
    fi
    payload=$(jq -n --arg content "$count new subdomains" '{content: $content}')
    curl -sS -H 'Content-Type: application/json' -d "$payload" "$DISCORD_WEBHOOK"
```

The same shape works for Slack. For Telegram, call `https://api.telegram.org/bot<token>/sendMessage`.

A missing webhook should not fail the run. Check for an empty value first.

### 7. Findings as issues

```yaml
permissions:
  contents: read
  issues: write

steps:
  - run: |
      while read -r host; do
        gh issue create --title "New host: $host" --body "https://$host"
      done < results/new-subdomains.txt
    env:
      GH_TOKEN: ${{ github.token }}
```

Issues give you a queue, labels, assignment, and a permanent record. Cheaper than a database and visible from a phone.

The token cannot trigger new workflow runs. That is intentional and prevents loops.

### 8. Artifacts or state

Use state for anything you need to compare across runs: subdomain lists, live host lists, previous scan results.

Use artifacts for large output you only need to read: raw nuclei JSON, screenshots, response dumps.

Set `retention-days` explicitly. Free accounts have 500 MB of artifact storage, shared with Packages.

### 9. Cadence

* Passive subdomain enumeration: daily.
* HTTP probing: after every enumeration.
* URL harvesting from Wayback and Common Crawl: weekly.
* Nuclei with updated templates: weekly, sharded.
* Port scanning: weekly and only where the program allows it.
* Manual single target: on demand.

Do not run everything at the same minute. Stagger schedules by 15 to 30 minutes.

### 10. Tuning

* `httpx -threads 50` is a reasonable default. Higher values trigger rate limits and false negatives.
* `nuclei -rl 150 -c 25` keeps load predictable.
* `subfinder -all` costs time. Use it daily only for small scopes.
* Avoid `-recursive` on large scopes in a daily job. Move it to a weekly job.
* Set `timeout-minutes` on every job. A hung tool should fail, not run for 6 hours.

## OPSEC and Cost

### Network identity

GitHub publishes its runner IP ranges at `https://api.github.com/meta`. Assume every target can see that you are on GitHub.

Consequences:

* WAFs and CDNs may block or challenge the whole range.
* Rate limits apply to the range, not only to your job.
* A scan that works from your VPS may return 403 everywhere from Actions.
* Results are not comparable between GitHub and a residential or VPS source.

This is a tradeoff, not a bug. GitHub gives you clean disposable machines and a scheduler. It does not give you quiet infrastructure.

### Attribution

Every run is tied to your account, repository, and commit. Logs are retained by GitHub and visible to anyone with repository access.

Assume a target can correlate requests to a GitHub account if it cares to. Do not use Actions for anything you would not attach your name to.

### Scope

Keep a `scope.txt` in the repository and check it before every scan.

```bash
if ! grep -qx "$TARGET" scope.txt; then
  echo "::error::$TARGET is not in scope.txt"
  exit 1
fi
```

A workflow that accepts a target input without a scope check is a liability. One mistyped domain is enough.

Read the program rules for:

* Whether automated scanning is allowed.
* Whether cloud providers are excluded.
* Rate limits and prohibited tools.
* Whether subdomains outside the listed scope are in scope.

### Terms

GitHub Actions is for work related to the repository. Recon for a program you are enrolled in fits. Mass scanning, proxy pools, or using runners as general compute does not.

Keep the repository private. A public repo exposes your target list, your results, and your history.

### Data handling

* No client data, credentials, or personal data in the repo.
* Keep artifacts short lived. Fourteen days is enough for most recon output.
* Do not commit response bodies or screenshots. Store host lists and summaries.
* Rotate any token or webhook that appears in a log.
* Delete the repository when an engagement ends.

### Cost

Private repositories draw from included compute. Public repositories are free but public.

As of late 2026:

* GitHub Free includes $12 of compute per month.
* A standard Linux runner costs $0.006 per minute.
* That is about 2,000 minutes per month on Free.
* Pro and Team include $18, about 3,000 minutes.
* Enterprise includes $300, about 50,000 minutes.

A daily 6 minute job is about 180 minutes per month, or $1.08. A weekly 60 minute scan across four shards is 240 minutes per month, or $1.44. Both fit inside Free with room to spare.

Storage is the constraint that surprises people:

* Free includes 500 MB of artifact storage, shared with Packages.
* Pro includes 1 GB, Team 2 GB, Enterprise 50 GB.
* Cache storage is 10 GB per repository and separate from artifacts.

A few thousand JSON lines are small. Screenshots are not. Keep retention short and upload only what you will read.

### When to move to a VPS

Use a VPS or a runner you host yourself when you need:

* Scans longer than 6 hours.
* A static IP for allowlisting.
* Inbound access, a listener, or a callback server.
* Persistent services such as a database, dashboard, or queue.
* Volume beyond the included compute.
* A target that blocks GitHub ranges.

A practical split: Actions for scheduling, passive enumeration, diffing, and small scans. VPS for long active scans, brute force, and anything that needs a stable network identity.

### Checklist

* Private repository.
* `scope.txt` checked before every run.
* `permissions` set to the minimum for each job.
* Secrets in repository or environment settings, never in YAML.
* Artifact retention set explicitly.
* No client data in the repo or artifacts.
* Budget alert configured on the account.
* Logs reviewed before the repository is shared or made public.

## Example: Subdomain Enum

### Goal

Take one domain, enumerate its subdomains, probe every host over HTTP, and produce txt files grouped by status class.

The workflow is in [`examples/recon.yml`](examples/recon.yml).

### Flow

```
subfinder dell.com  ->  results/subdomains.txt
        |
      httpx -json   ->  results/httpx.jsonl
        |
      jq split      ->  status.txt, 2xx.txt, 3xx.txt, 4xx.txt, 5xx.txt
```

### Walkthrough

#### Trigger and scope

```yaml
on:
  workflow_dispatch:
  schedule:
    - cron: '17 3 * * *'

env:
  TARGET: dell.com
```

`workflow_dispatch` gives you a Run workflow button. The cron runs daily at 03:17 UTC. The minute is deliberate: the top of the hour is the busiest queue.

`TARGET` is the only value you change to point this at another domain.

#### Permissions and concurrency

```yaml
permissions:
  contents: read

concurrency:
  group: recon-${{ github.ref }}
  cancel-in-progress: false
```

The job only reads the repository, so `contents: read` is enough.

The concurrency group prevents two runs from stacking. `cancel-in-progress: false` means a new run waits instead of killing a scan that is already working.

#### Tool install

```yaml
- uses: actions/setup-go@v7
  with:
    go-version: 'stable'

- uses: actions/cache@v6
  with:
    path: |
      ~/go/pkg/mod
      ~/.cache/go-build
    key: go-tools-${{ runner.os }}-subfinder-2.17.0-httpx-1.12.0

- run: |
    go install github.com/projectdiscovery/subfinder/v2/cmd/subfinder@v2.17.0
    go install github.com/projectdiscovery/httpx/cmd/httpx@v1.12.0
    echo "$(go env GOPATH)/bin" >> "$GITHUB_PATH"
```

Versions are pinned. The cache key includes both versions, so bumping a tool creates a new cache instead of restoring a stale one.

The first run compiles the tools and takes a few minutes. Later runs reuse the module and build cache.

#### Enumeration

```yaml
- run: |
    mkdir -p results
    subfinder -d "$TARGET" -all -silent -o results/subdomains.txt
    count=$(wc -l < results/subdomains.txt)
    if [ "$count" -eq 0 ]; then
      echo "::error::subfinder returned no subdomains"
      exit 1
    fi
```

`-all` uses every source that is available. Without API keys, several sources are skipped, so results are thinner than a configured install.

The zero check matters. An empty list would make the next step succeed with no work and hide the real problem.

#### Probing

```yaml
- run: |
    httpx -l results/subdomains.txt \
      -silent \
      -threads "$HTTPX_THREADS" \
      -timeout 10 \
      -retries 1 \
      -json \
      -o results/httpx.jsonl
```

`-json` writes one object per host. That object is what the reporting step reads. `-threads 50` is a safe default for most targets.

httpx probes HTTPS first, then HTTP. A host that answers with 403 or 500 is still a live host and still appears in the output.

#### Reports

```yaml
- run: |
    jq -r '"\(.status_code)\t\(.url)\t\(if (.title // "") == "" then "-" else .title end)"' \
      results/httpx.jsonl | sort -n > results/status.txt

    for class in 2 3 4 5; do
      jq -r --argjson lo "${class}00" --argjson hi "$((class + 1))00" \
        'select(.status_code >= $lo and .status_code < $hi) | "\(.status_code)\t\(.url)"' \
        results/httpx.jsonl | sort -n > "results/${class}xx.txt"
    done
```

`status.txt` is the human readable view: code, URL, title. The per class files are for triage and for feeding later tools.

Tabs separate the fields, so `cut -f2` gives you a clean URL list.

#### Artifact and summary

```yaml
- uses: actions/upload-artifact@v7
  if: always()
  with:
    name: recon-${{ github.run_number }}
    path: results/
    retention-days: 14
```

`if: always()` keeps partial results when an earlier step fails. That is usually when you need them most.

The job summary writes `summary.txt` to the run page, so you can read counts without downloading anything.

### Output

* `results/subdomains.txt`: every subdomain subfinder returned
* `results/httpx.jsonl`: raw httpx output, one JSON object per host
* `results/status.txt`: status code, URL, page title
* `results/2xx.txt`: hosts returning 200 to 299
* `results/3xx.txt`: hosts returning 300 to 399
* `results/4xx.txt`: hosts returning 400 to 499
* `results/5xx.txt`: hosts returning 500 to 599
* `results/summary.txt`: counts for the run

### Run it

Copy the example into a private repository and push it:

```bash
mkdir -p ~/bb-recon/.github/workflows
cp examples/recon.yml ~/bb-recon/.github/workflows/recon.yml
cd ~/bb-recon
git init -b main
git add .
git commit -m "recon workflow"
gh repo create bb-recon --private --source=. --push

gh workflow run recon.yml
gh run watch
gh run download --dir ./reports
```

The artifact lands in `./reports/recon-<run_number>/`.

### Tuning

* Change `TARGET` to another domain.
* Raise `HTTPX_THREADS` only after watching for 429 and 403 spikes.
* Add `-recursive` to subfinder for deeper enumeration, at the cost of time.
* Add provider keys for Shodan, Censys, VirusTotal, and SecurityTrails to improve coverage.
* Lower `timeout-minutes` if you want a stuck run to fail sooner.

### Extending

The same shape scales to the rest of the stack:

* Add `dnsx` between subfinder and httpx to drop hosts that do not resolve.
* Add `naabu` to find open ports, then feed the results into httpx.
* Add `nuclei` with `-severity critical,high` and shard it with a matrix.
* Add `katana` or `gau` for URL discovery, then filter with `gf` patterns.

Patterns for sharding, state, and notifications are in [Bug Bounty Patterns](#bug-bounty-patterns).

### Failure modes

* Subfinder returns zero: sources are blocked or rate limited. Add API keys and rerun.
* httpx returns zero: DNS is failing or every host is unreachable. Check `subdomains.txt` first.
* First run is slow: Go is compiling both tools. Later runs use the cache.
* No artifact: the upload step is set to `always()`, so check the log for an earlier failure.
* Workflow stops running on schedule: the repository was inactive for 60 days. Open the Actions tab and enable it again.
* Everything returns 403: the target blocks GitHub ranges. Move that scan to a VPS.

### Scope

This example targets `dell.com`. Run it only if that domain is in scope for you, and keep the repository private. The workflow is a template, not an authorization.

## Repository Map

```text
.
├── README.md
├── examples/
│   └── recon.yml
└── .gitignore
```
