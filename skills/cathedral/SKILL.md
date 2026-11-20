---
name: cathedral
description: Run code in Cathedral confidential sandboxes (AMD SEV-SNP) from the command line with ctcli. Use when you need an isolated Linux sandbox to run commands, tests, evals, RL rollouts or untrusted code; when you need to move files in and out, fork a running sandbox, run background jobs, run Harbor evals, or run GitHub Actions jobs in a confidential sandbox; or when you need receipts you can verify offline against AMD's certificate chain.
---

# Cathedral confidential sandboxes

A Cathedral sandbox is a Linux room on an AMD SEV-SNP machine. SEV-SNP encrypts the machine's memory so the host cannot read it, and ctcli encrypts your commands and files on your machine to the attested guest before it sends them. Every create and every command leaves receipts you can check offline.

This skill drives everything with `ctcli`. The HTTPS contract behind it is https://cathedral.computer/agents.md. Cathedral is in live testing.

## When to use

- You need a disposable Linux sandbox to run commands, tests, builds or untrusted code.
- You run evals, graders or RL rollouts and want one sandbox per trial, with a receipt chain per trial.
- You want to snapshot a prepared sandbox and fork copies of it.
- You want GitHub Actions jobs to run in confidential sandboxes (CI, open to every team).

## What it costs

Billed by the second while a sandbox exists. Unused prepaid time is refunded on delete. No plan fee.

| Size | vCPU | Memory | Disk | Per hour |
| --- | --- | --- | --- | --- |
| `sealed-1x2` | 1 | 2 GiB | 10 GiB | $0.005 |
| `sealed-2x4` | 2 | 4 GiB | 10 GiB | $0.01 |
| `sealed-4x8` | 4 | 8 GiB | 10 GiB | $0.02 |
| `sealed-8x16` | 8 | 16 GiB | 10 GiB | $0.04 |

That is $0.0025 per GiB-hour at every size. Live prices: `curl -s https://cathedral.computer/v1/profiles` (no key).

New accounts get a one-time $20 trial credit the first time your person opens https://cathedral.computer/console/ while signed in. Approving a login alone does not add it.

Same shapes elsewhere, compute and memory only, from each vendor's public pricing page on 18 Nov 2026:

| Per hour | Cathedral | E2B | Daytona |
| --- | --- | --- | --- |
| 2 vCPU, 4 GiB | $0.01 | $0.1656 | $0.1656 |
| 8 vCPU, 16 GiB | $0.04 | not listed | $0.6624 |
| Billing | per second | per second | per second |
| Free credit | $20 once | $100 once (Hobby) | $200 included |

E2B: $0.000014 per vCPU-second plus $0.0000045 per GiB-second, memory listed up to 8 GiB, custom sizes on Pro at $150 a month (https://e2b.dev/pricing). Daytona: $0.0504 per vCPU-hour plus $0.0162 per GiB-hour, storage above 5 GiB extra (https://www.daytona.io/pricing). Check those pages before you quote them; prices change.

## Rules

- Never print, log or commit a key. Never put one in a command argument, a label or a URL.
- Ask your person before you create sandboxes, and agree a budget first.
- Label every sandbox with the job (`--label job=<job>`). Delete every sandbox you create, including after errors.
- Use `--template` explicitly. Pass `--` before the remote command.
- If the API says 402 `insufficient_credits`, stop and ask your person to add credit at https://cathedral.computer/account/. A new account should first open https://cathedral.computer/console/ once, which adds the trial credit. You cannot pay.

## 1. Install ctcli

```sh
curl -fsSL https://cathedral.computer/install.sh | sh     # macOS or Linux
ctcli --version                                           # ctcli 0.6.1
```

Other ways: `brew install cathedralai/tap/ctcli` (macOS), `irm https://cathedral.computer/install.ps1 | iex` (Windows PowerShell), or `uv tool install https://cathedral.computer/ctcli/ctcli-0.6.1-py3-none-any.whl`. With pip, two steps: `python3 -m pip install --user "cryptography>=42"`, then `python3 -m pip install --user --no-deps --index-url https://cathedral.computer/ctcli/simple/ ctcli`, and never `--extra-index-url`: the `ctcli` on PyPI is a different project.

## 2. Connect: your person approves a budget

Run the device login in the background. It blocks up to 10 minutes and prints one JSON line on stderr with the approval link.

```sh
ctcli --json login --api-base https://cathedral.computer --budget 25 --days 30 2>login.err &
sleep 3; cat login.err
# {"event": "login_approval_required", "verification_url": "https://cathedral.computer/cli-login?code=ABCD-EFGH", "user_code": "ABCD-EFGH", ...}
```

- Send your person the `verification_url` and `user_code`. The page shows what the key can do, its budget and its expiry; they can lower both. If they have no account, the same page signs them up (Google or an email link).
- The key it mints can create, run and delete sandboxes and read the balance. It cannot buy credit. It stops at its budget (402 `key_budget_exceeded`) and at its expiry.
- ctcli saves the key 0600 at `~/.config/cathedral/ctcli.json` and never prints it. A key you already have: `printf '%s' "$CATHEDRAL_API_KEY" | ctcli login --key`.
- Check your room limit and the catalog:

```sh
ctcli sandbox quota
ctcli sandbox catalog
```

## 3. Create a confidential sandbox

```sh
ctcli --json sandbox create --template sealed-2x4 --lifetime 3600 --label job=myjob --wait > sbx.json
SBX=$(python3 -c 'import json;print(json.load(open("sbx.json"))["id"])')
ctcli sandbox status "$SBX"
```

- Defaults: network off (`deny_all`), and a spend cap ctcli computes from the rate. Set your own with `--max-spend-usd`.
- `--network internet`, or `--network allowlist --egress-allow pypi.org` (at most 32 hosts).
- `--lifetime` is in seconds, at most 86400. Extend a running sandbox: `ctcli sandbox extend "$SBX" --seconds 600`.
- Variables: `--env-var` and `--env-file` at create are refused by default (`sealed_env_at_create_not_ready`), because they are not encrypted to the guest yet. Pass them per command with `ctcli sandbox exec "$SBX" --env NAME=VALUE -- ...` (up to 64, encrypted). Adding `--unsealed` to the create sends them in plaintext through Cathedral's API.
- Check `"hardware": "amd_sev_snp"` in the answer. If it says anything else, delete the sandbox and tell your person.
- If the create answer is lost, `ctcli sandbox recover <request_id>` (printed on stderr) finishes the same create without a second sandbox.
- Sold out (503 `sealed_capacity_unavailable` and similar): nothing was charged. Wait `Retry-After`, then create again.

## 4. Run commands

```sh
ctcli sandbox exec "$SBX" -- python3 -c 'print(6 * 7)'
ctcli sandbox exec "$SBX" -- sh -c 'cd /work && ls -la'
```

- A foreground command runs at most 45 seconds and is encrypted to the attested guest. ctcli exits with the command's exit code.
- The command is argv. Use `sh -c '...'` for shell syntax.
- There is no SSH and no interactive shell.

## 5. Upload and download files

```sh
ctcli sandbox upload "$SBX" ./job.py /work/job.py
ctcli sandbox exec "$SBX" -- python3 /work/job.py
ctcli sandbox download "$SBX" /work/out.txt -o out.txt
```

- Use paths under `/work`. Copy out what you keep before you delete.
- `upload` moves one file of up to 256 MiB, in encrypted chunks. `download` refuses files over 10 MiB unless you pass `--max-bytes` (at most 256 MiB; 200 MiB with `--unsealed`). For bigger files, `split -b 90M`, move the parts, and join them with `cat` inside the sandbox.

## 6. Background jobs (more than 45 seconds)

Background commands are not encrypted to the guest yet: their command and output pass through Cathedral's API in plaintext, so ctcli asks for `--unsealed`. Keep secrets out of them.

```sh
EXEC=$(ctcli --unsealed sandbox exec "$SBX" --timeout 3600 --detach -- sh -c 'sleep 60; echo done' 2>/dev/null)
ctcli --unsealed sandbox exec-status "$SBX" "$EXEC" --wait     # prints the output, exits with its exit code
```

- `--timeout` is 1 to 14400 seconds; the default is 60, and a command that runs longer is stopped (exit 124). Set it on every background job.
- Without `--detach`, a `--timeout` above 45 runs the command in the background and waits for it.
- `ctcli --unsealed sandbox exec-cancel "$SBX" "$EXEC"` stops it.

## 7. Snapshot and fork

A snapshot saves a running sandbox's disk (its image changes and `/work`) on its machine. Running processes and memory are not kept: start services again in each fork. Each fork is a new sandbox from the snapshot, billed like any other.

```sh
ctcli --json sandbox snapshot create "$SBX" --wait --wait-seconds 60 > snap.json 2>snap.err
SNAP=$(python3 -c 'import json;d=json.load(open("snap.json"));print(d["id"]) if d["state"]=="ready" else exit(1)')
ctcli --json sandbox create --snapshot "$SNAP" --max-spend-usd 0.05 --label job=myjob --wait > fork.json 2>fork.err
FORK=$(python3 -c 'import json;print(json.load(open("fork.json"))["id"])')
ctcli sandbox exec "$FORK" -- ls /work      # the source's files are there
```

- A fork keeps the source's size and machine; `--max-spend-usd` is required.
- Delete snapshots when done: `ctcli sandbox snapshot rm "$SNAP" --yes`. `ctcli sandbox snapshot ls` lists them. A snapshot expires after 24 hours by default (`--ttl`, at most 7 days).
- If the API answers 422 `unsupported_on_sealed_pool`, snapshots are off for your account. If a snapshot stays `creating` for more than a minute (2 of 9 did in our tests on 18 Nov 2026), remove it and take a new one.

## 8. Verify receipts

Each command leaves two receipts: a platform receipt from Cathedral's API, and a guest receipt signed inside the confidential guest by a key that the machine's AMD SEV-SNP attestation report commits to. Check both offline:

```sh
ctcli sandbox evidence "$SBX" -o evidence.json
curl -fsSO https://cathedral.computer/customer-receipt-trusted-keys.json
curl -fsSO https://cathedral.computer/approved-measurements.json
ctcli receipt verify --bundle evidence.json \
  --trusted-keys-file customer-receipt-trusted-keys.json \
  $(python3 -c 'import json;print(" ".join("--approved-measurement "+m["measurement"] for m in json.load(open("approved-measurements.json"))["measurements"]))')
```

- `PASS` (exit 0): the SEV-SNP report and AMD's ARK, ASK and VCEK chain verified with debug off, the measurement is on the approved list, every guest receipt verified, the platform receipt chain verified against the pinned key, and each command matched its guest receipt.
- `FAIL` (exit 1) or `NOT PROVEN` (exit 3): the output names the line that failed. Report it; do not call the result verified.
- Pin the two downloaded files through a channel you trust before you rely on them. The approved list is Cathedral's list of the guest builds it runs; those builds are not yet reproducible from source outside Cathedral.
- A receipt proves which command ran where, not that your code is correct.
- ctcli 0.6.1 accepts the `claim` guest receipt of a sandbox that starts from a pre-made room and the `exec_cancel` receipt of a cancelled command. ctcli 0.6.0 answers FAIL on both: update ctcli, and report such a FAIL from 0.6.0 as that gap, not as a failed run.

## 9. Delete

```sh
ctcli sandbox delete "$SBX" --yes
ctcli sandbox delete --label job=myjob --yes      # every live sandbox of the job
ctcli sandbox list
ctcli sandbox usage --group-by job --label job=myjob
```

Billing stops when the delete is accepted. Report one line per job: sandboxes created, failed, still running, and the cost from `usage`.

## 10. Harbor evals

Harbor runs each trial in its own confidential sandbox through the `cathedral-harbor` provider (Python 3.12 or newer, Harbor 0.21).

```sh
python3 -m venv .venv
.venv/bin/pip install --index-url https://cathedral.computer/harbor/simple/ cathedral-harbor
export CATHEDRAL_API_KEY="$(python3 -c 'import json,os;print(json.load(open(os.path.expanduser("~/.config/cathedral/ctcli.json")))["auth"]["api_key"])')"
.venv/bin/harbor run -d terminal-bench@2.0 -i chess-best-move -a oracle \
  -e cathedral_harbor:CathedralEnvironment -n 1
```

- `-n` is how many trials run at once. Each trial's sandbox is labelled with the Harbor job id and deleted when the trial ends.
- Settings: `CATHEDRAL_TEMPLATE` (size), `CATHEDRAL_LIFETIME_SECONDS`, `CATHEDRAL_MAX_SPEND_USD`, `CATHEDRAL_JOB_LABEL`.
- A task needs a prebuilt registry image; a sandbox create does not build a Dockerfile (one that is only `FROM` and `WORKDIR` runs on its `FROM` image).
- The oracle agent replays the reference solution, so its reward is not a model score.

## 11. CI: GitHub Actions in confidential sandboxes

Each job runs in its own confidential sandbox, billed by the second. Your person sets it up in the console; an API key cannot.

CI is open to every team: up to 2 jobs at once and 25 USD a month per team. Ask us if you need more.

1. Your person opens https://cathedral.computer/console/ci/ in a team.
2. A Cathedral team owner or admin, who is also an owner or admin of the GitHub organization or account, presses "Connect GitHub", installs the Cathedral CI app and picks private repositories. Public repositories are refused for now: their jobs are never picked up.
3. You put one Cathedral label in `runs-on`, and nothing else:

```yaml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: cathedral-4x8      # or cathedral-2x4, cathedral-8x16
    steps:
      - uses: actions/checkout@v4
      - run: make test
```

Labels: `cathedral-2x4` (2 vCPU, 4 GiB), `cathedral-4x8` (4 vCPU, 8 GiB), `cathedral-8x16` (8 vCPU, 16 GiB).

## What confidential covers, and what it does not

- Covered: the machine's memory is encrypted by AMD SEV-SNP. Cathedral checks each machine's attestation when it admits it, and publishes it. ctcli checks the attestation on your machine and encrypts foreground commands, their output, and uploads and downloads to the guest's key; Cathedral's edge, API and broker carry ciphertext.
- Not covered: who called, when, how often and how large; background jobs, processes, env values set at create, labels and anything sent with `--unsealed`; the guest image, which Cathedral builds. Cathedral's API and broker can send commands of their own into your room; each one appears in the guest receipt chain. Several customers' rooms share one machine, and the room runtime, not the hardware, separates them.
- Keep secrets you cannot share with Cathedral out of the sandbox. Full list: https://cathedral.computer/docs/sandboxes/what-cathedral-can-see/

## Reference

- Agent contract (HTTPS): https://cathedral.computer/agents.md
- Docs index: https://cathedral.computer/llms.txt
- Errors: https://cathedral.computer/docs/help/sealed-errors/
- Every ctcli command: `ctcli sandbox --help`, `ctcli sandbox <command> --help`
- Compute providers and miners on Bittensor SN94: https://cathedral.computer/miners.md
