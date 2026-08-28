# Git hooks

A small, dependency-light set of Git hooks for this repo:

- **`precommit`** — auto-formats staged files by extension and runs several
  safety guards before a commit is allowed through.
- **`prepare-commit-msg`** — seeds the commit message from a `.gitmessage`
  template and substitutes branch/ticket placeholders.
- **`.gitmessage`** — the commit-message template used by `prepare-commit-msg`.

> Requires **Bash ≥ 4.0** (for associative arrays). macOS ships Bash 3.2 —
> install a newer one with `brew install bash`.

## Install

From the repo root:

```bash
cp hooks/precommit            .git/hooks/pre-commit        && chmod +x .git/hooks/pre-commit
cp hooks/prepare-commit-msg   .git/hooks/prepare-commit-msg && chmod +x .git/hooks/prepare-commit-msg
```

(Optionally) let Git load the template itself, so editors open with the
template even outside this hook:

```bash
git config commit.template hooks/.gitmessage
```

### One-line installer

```bash
for h in precommit:pre-commit prepare-commit-msg:prepare-commit-msg; do
  src=${h%%:*}; dst=${h##*:}
  cp "hooks/$src" ".git/hooks/$dst" && chmod +x ".git/hooks/$dst"
done
git config commit.template hooks/.gitmessage
```

## `precommit`

Runs in this order, all reading only from the index (a rejected commit never
touches your working tree):

1. **Protected-branch guard** — refuses direct commits to `main`, `master`, or
   `release/*`. Use a feature branch / pull request instead. Only enforced
   when more than one branch exists, so a fresh repo with only its initial
   branch isn't blocked.
2. **Large-file guard** — rejects staged blobs larger than 1 MiB.
3. **Secrets scan** — scans staged content for common secret patterns (AWS
   keys, private keys, GitHub/Google/Slack tokens, generic
   `api_key=`/`secret=`/`password=`/`token=` assignments) and asks yes/no.
   In a non-interactive context it warns and proceeds rather than hanging.
4. **Stash unstaged changes** (so `git add -p` partial staging is preserved),
   then per-extension formatting:
   - **Go**: `go fix` → `gofmt` → `go vet`
   - **Rust**: `rustfmt` · **Python**: `black` (or `ruff format`)
   - **JS/TS/JSON/YAML/Markdown/CSS**: `prettier`
   - **C/C++/Obj-C/Protobuf**: `clang-format` (or `buf format` for `.proto`)
   - **Java**: `google-java-format` · **Shell**: `shfmt` · **Terraform**: `terraform fmt`
5. Files with unresolved merge/rebase conflict markers are skipped by default
   (not formatted); the commit still proceeds.

### Configuration (environment variables)

| Variable | Default | Meaning |
| --- | --- | --- |
| `PRECOMMIT_PROTECTED_BRANCHES` | `main master release/*` | Space-separated glob patterns of protected branches |
| `PRECOMMIT_ALLOW_PROTECTED` | `0` | `1` bypasses the protected-branch guard (e.g. intentional merges) |
| `PRECOMMIT_MAX_FILE_SIZE` | `1048576` (1 MiB) | Max staged blob size; accepts `K`/`M`/`G` suffixes. `0` disables |
| `PRECOMMIT_SKIP_SECRETS` | `0` | `1` silences the secrets scan entirely (no warning, no prompt) |
| `PRECOMMIT_VET_FATAL` | `0` | `1` makes `go vet` findings block the commit (off = warnings only) |
| `PRECOMMIT_BLOCK_CONFLICTS` | `0` | `1` refuses to commit while conflict markers are present (default: skip & allow) |

Examples:

```bash
PRECOMMIT_ALLOW_PROTECTED=1 git commit -m "merge main"
PRECOMMIT_MAX_FILE_SIZE=5M git commit ...
PRECOMMIT_VET_FATAL=1 git commit ...
```

## `prepare-commit-msg`

Looks up `.gitmessage` in this order: next to the script, repo root, repo
`hooks/`, `$GIT_DIR`, `$HOME`. Substitutes these placeholders:

- `{{BRANCH}}` → current branch name (e.g. `feature/ABC-123-login`)
- `{{TICKET}}` → ticket id parsed from the branch (e.g. `ABC-123`)

Behavior by commit source:

- **Editor** (`git commit`, empty message): prepends the rendered template
  above Git's status comments.
- **`git commit -m "…"`**: left untouched unless `PRECOMMIT_INJECT_TICKET=1`,
  which prefixes the subject in place (`feat: x` → `ABC-123: feat: x`).
- **`commit.template` already set** (`source=template`): only substitutes
  placeholders in Git's loaded template.
- **merge / squash**: left untouched.

### Configuration

| Variable | Default | Meaning |
| --- | --- | --- |
| `PRECOMMIT_INJECT_TICKET` | `0` | `1` auto-prefixes `-m` subjects with the branch's ticket id |

## `.gitmessage`

A commented Conventional-Commits template with `{{BRANCH}}` / `{{TICKET}}`
placeholders. Edit it to taste — e.g. replace the blank line with
`{{TICKET}}: <subject>` to have the ticket auto-placed in the subject for
editor commits.
