# Cathedral skill

An agent skill for [Cathedral](https://cathedral.computer) confidential sandboxes: Linux rooms on AMD SEV-SNP machines, driven from the command line with `ctcli`.

The skill is one file, [`skills/cathedral/SKILL.md`](skills/cathedral/SKILL.md). The same file is served at https://cathedral.computer/skill.md.

It covers: installing ctcli, connecting by device login (your person approves a budget), creating a confidential sandbox, running commands, moving files, background jobs, snapshot and fork, verifying receipts offline, deleting, Harbor evals and CI on GitHub Actions. Every command in it was run on production before it was published.

## Install

Claude Code (personal skills):

```sh
mkdir -p ~/.claude/skills/cathedral
curl -fsSL https://cathedral.computer/skill.md -o ~/.claude/skills/cathedral/SKILL.md
```

Claude Code (one project): put the file at `.claude/skills/cathedral/SKILL.md` in the repository.

Any other agent: point it at https://cathedral.computer/skill.md, or paste the file into its instructions.

## Price

$0.0025 per GiB-hour of memory at every size ($0.01 an hour for 2 vCPU and 4 GiB), billed by the second. Live prices: https://cathedral.computer/v1/profiles

## More

- Agent contract over HTTPS: https://cathedral.computer/agents.md
- Docs: https://cathedral.computer/docs/
- What confidential covers: https://cathedral.computer/docs/sandboxes/what-cathedral-can-see/
