# Claude Code Practices
**Meeting Presentation — Computational Biology & Algorithm Development**

---

## 1. Setting Up `CLAUDE.md`

### What It Is
`CLAUDE.md` is a persistent instruction file Claude Code reads **automatically** at the start of every session — your standing orders, never re-explained.

### Where to Put It

| Location | Scope |
|---|---|
| `~/.claude/CLAUDE.md` | Global — applies to ALL projects |
| `<project-root>/CLAUDE.md` | Project-level — committed to repo, shared with team |
| `<subdirectory>/CLAUDE.md` | Scoped — only when working inside that folder |

> Claude merges all applicable files: Global → Project → Subdirectory.

---

### Template A — Computational Biologist (Multi-Omics Analysis)

For researchers running pipelines, integrating omics datasets, and producing publication figures.

```markdown
# Project: <Study Name / Cohort>

## Role & Context
You are assisting a computational biologist working on multi-omics integration
(e.g., bulk/single-cell RNA-seq, ATAC-seq, proteomics, metabolomics).
The goal is rigorous, reproducible analysis leading to publication-quality results.

## Tech Stack
- Primary languages: R, Python
- Workflow manager: Script of Scripts (SoS)
- Key R packages: Seurat, DESeq2, edgeR, limma, ggplot2, ComplexHeatmap
- Key Python packages: scanpy, anndata, pandas, matplotlib, scikit-learn
- Environment management: pixi, conda (environment.yml committed to repo)

## Code Style
- R: snake_case everywhere; tidyverse style; no `<-` vs `=` mixing (use `<-`)
- Python: snake_case variables, PascalCase classes; Black-formatted
- No inline comments unless the statistical logic is non-obvious
- Every analysis script must have a header block:
  # Author, Date, Purpose, Input, Output

## Reproducibility Rules
- Set seeds explicitly: `set.seed(42)` / `random.seed(42)` at script top
- Never hardcode absolute paths — use `here::here()` in R or `pathlib` in Python
- All figures saved to `results/figures/` with explicit DPI (≥300 for publication)
- Raw data is read-only — never overwrite files in `data/raw/`
- Intermediate results go to `data/processed/` with descriptive filenames

## Workflow Rules
- Always read a file before editing it
- Prefer editing existing scripts over creating new ones
- Run tests / sanity checks (sample counts, NA rates, dimension checks) before
  declaring an analysis step complete
- Do not push to `main` directly — use feature branches named `analysis/<topic>`

## Forbidden Actions (ask first)
- `git push --force`
- Overwriting files in `data/raw/`
- Deleting intermediate results without confirmation
- Changing statistical parameters (normalization method, thresholds) without noting it

## Useful Commands
- Run pipeline: `snakemake --cores 8` / `sos run pipeline.sos`
- Render report: `Rscript -e "rmarkdown::render('analysis/report.Rmd')"`
- Activate env: `conda activate <env-name>`
- Lint R: `Rscript -e "lintr::lint('script.R')"`

## Project-Specific Notes
- Sample metadata: `data/raw/metadata.csv` (never modify directly)
- Genome reference: hg38 / mm10 — confirm before any alignment step
- Cluster credentials / HPC config in `~/.ssh/config` (not committed)
```

---

### Template B — Algorithm Developer (Biological Methods / Bioinformatics Tools)

For developers building new computational methods, tools, or benchmarking frameworks targeting biological problems.

```markdown
# Project: <Tool / Algorithm Name>

## Role & Context
You are assisting a bioinformatics algorithm developer building a computational
method for biological data analysis (e.g., dimensionality reduction, clustering,
multi-omics integration, variant calling, network inference).
The goal is a well-tested, documented, and publishable software tool.

## Tech Stack
- Primary languages: Python (core algorithm), R (wrapper / Bioconductor package)
- Build / packaging: pyproject.toml (Python) or DESCRIPTION + NAMESPACE (R package)
- Testing: pytest (Python) / testthat (R)
- Docs: Sphinx / pkgdown
- Benchmarking: snakemake benchmarking module or custom harness

## Code Style
- Python: snake_case variables, PascalCase classes; type-annotated function
  signatures; Black + isort formatted; docstrings in NumPy style
- R: snake_case; roxygen2 docstrings on all exported functions
- Algorithmic complexity noted in docstring when non-trivial (O(n log n), etc.)
- No magic numbers — define as named constants with explanatory comments

## Software Engineering Rules
- Always read a file before editing it
- Write unit tests for every new function before or alongside implementation
- All public API changes must update the changelog (CHANGELOG.md)
- Prefer composition over inheritance; keep functions ≤ 50 lines
- Do not push to `main` directly — use branches: `feat/`, `fix/`, `bench/`
- Version-bump follows semantic versioning (MAJOR.MINOR.PATCH)

## Testing & Benchmarking
- Unit tests: `pytest tests/` / `Rscript -e "devtools::test()"`
- Benchmark datasets stored in `tests/data/` (small, synthetic, committed)
- Real benchmark datasets referenced by path in `config/benchmark_paths.yaml`
  (not committed — too large)
- Performance regression: flag if runtime increases >10% vs previous release

## Forbidden Actions (ask first)
- `git push --force`
- Breaking changes to public API without major version bump
- Removing or renaming exported functions without deprecation notice
- Committing large binary files (>1 MB) to the repo

## Useful Commands
- Install dev: `pip install -e ".[dev]"` / `Rscript -e "devtools::install()"`
- Run tests: `pytest --cov=src tests/` / `Rscript -e "devtools::test()"`
- Build docs: `make html` (Sphinx) / `Rscript -e "pkgdown::build_site()"`
- Lint: `ruff check src/` / `Rscript -e "lintr::lint_package()"`

## Project-Specific Notes
- Algorithm paper draft: `docs/manuscript/` — sync notation with code variable names
- Preprint DOI / companion dataset: record in README under "Citation"
- Conda env for reproducible benchmarks: `envs/benchmark.yaml`
```

---

## 2. Git as a Security Checkpoint

### Core Concept
Use Git hooks as **mandatory review gates** — Claude respects them and cannot silently bypass them.

---

### Auto Git Setup — From Scratch (HPC + GitHub)

#### Step 1 — Generate SSH key (once per machine)
```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
# Accept default path (~/.ssh/id_ed25519), set passphrase or leave empty
```

#### Step 2 — Copy public key and add to GitHub
```bash
cat ~/.ssh/id_ed25519.pub
# Copy the output, then:
# GitHub → Settings → SSH and GPG keys → New SSH key → paste & save
```
Verify the fingerprint matches:
```bash
ssh-keygen -lf ~/.ssh/id_ed25519.pub
# Should match what GitHub shows under your SSH key entry
```

#### Step 3 — Create `~/.ssh/config`
```bash
# ~/.ssh/config
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519
  AddKeysToAgent yes
```
```bash
chmod 600 ~/.ssh/config
```

> **HPC note:** If `~/.ssh` lives on a shared filesystem (e.g., Lustre), the key is accessible from **all cluster nodes** — no per-node setup needed. The key comment (`root@hostname`) is a label only; it does not restrict where the key works.

#### Step 4 — Switch remote from HTTPS to SSH
```bash
git remote set-url origin git@github.com:<user>/<repo>.git
git remote -v   # confirm URL starts with git@
```

#### Step 5 — Test
```bash
ssh -T git@github.com
# Expected: "Hi <username>! You've successfully authenticated..."
git push   # should now work without password prompts
```

---

### Git Conventions for Rollback (Add to `CLAUDE.md`)

In Claude Code, Git is not just version control — it is the **safety net between you and Claude**.
The faster Claude executes, the finer-grained this net must be.
Fine-grained commits + clear messages + explicit checkpoints = precise, low-cost rollback.

#### Commit Message Format (Conventional Commits)

```
<type>(<scope>): <subject — ≤72 chars>

<body — explain why, trade-offs, and known limitations>
```

| Type | Meaning |
|---|---|
| `feat` | New feature |
| `fix` | Bug fix |
| `refactor` | Refactoring (no functional change) |
| `perf` | Performance improvement |
| `test` | Test-related changes |
| `docs` | Documentation only |
| `checkpoint` | Safety snapshot before a large change |

**Pre-commit checklist:**
- Code compiles / script runs without error
- Relevant unit tests pass
- No debug `print` / `cat` statements committed

**Good commit body example:**
```
feat(keyframe): add scene change detection using histogram diff

Chose histogram approach over ML model because:
- Single-frame processing < 5ms on target hardware
- No extra model files needed (~30MB savings)
- Error rate < 3%, sufficient for current use case

Known limitation: HDR video histograms differ in distribution;
detection threshold needs tuning — HDR not supported in this release.
```

**Bad commits (useless for rollback):**
```
fix bug
update code
WIP
```

#### Commit Granularity

Commit once per meaningful step — do **not** wait for an entire feature to be done.
Fine-grained commits let Claude `git revert` a single step precisely.

```bash
# After completing Step 1
git commit -m "feat(pipeline): add sample QC filter by read depth"

# After completing Step 2
git commit -m "feat(pipeline): add normalization using DESeq2 size factors"
```

#### Branch Naming

| Scenario | Branch name |
|---|---|
| New feature | `feature/<short-description>` |
| Bug fix | `fix/<issue-description>` |
| Refactor | `refactor/<module-name>` |
| Analysis run | `analysis/<dataset>-<assay>` |

#### Checkpoint Before Any Large Change

Before asking Claude to do any broad refactor or multi-file change, manually commit a checkpoint:

```bash
git add -A && git commit -m "checkpoint: before refactoring <module-name>"
```

This is your guaranteed recovery point — no matter what Claude does next, you can always return here with:

```bash
git reset --hard <checkpoint-commit-hash>
```

#### Rollback Commands

```bash
# Find the offending commit
git log --oneline -20

# Safe rollback: revert one commit without touching others
git revert <commit-hash>

# Full reset to checkpoint (destructive — discards all commits after it)
git reset --hard <checkpoint-commit-hash>
```

> **Rule:** The better your commit messages, the more precisely Claude can locate and revert the exact change that caused a problem — without touching anything else.

---

### Auto-push Behavior (Add to `CLAUDE.md`)
```markdown
## Auto-push Rule
When a task is complete and changes are staged:
1. `git status` — confirm no raw data files staged
2. `git push` — if auth fails, run `ssh-add ~/.ssh/id_ed25519` and retry once
3. If still failing, report the exact error; never store or echo credentials
```
---

### Pre-push Security Hook

Install once per repo — Claude will respect it and never use `--no-verify`.

```bash
# Save as .git/hooks/pre-push, then: chmod +x .git/hooks/pre-push
#!/bin/bash

echo "=== Pre-push security check ==="

# Check if running in interactive terminal
IS_INTERACTIVE=false
if [ -t 0 ] && [ -e /dev/tty ]; then IS_INTERACTIVE=true; fi

# Warn on direct push to main/master
BRANCH=$(git symbolic-ref HEAD | sed 's|refs/heads/||')
if [[ "$BRANCH" == "main" || "$BRANCH" == "master" ]]; then
  if $IS_INTERACTIVE; then
    echo "WARNING: Pushing directly to $BRANCH. Continue? [y/N]"
    read -r answer < /dev/tty
    [[ "$answer" == "y" || "$answer" == "Y" ]] || exit 1
  else
    echo "WARNING: Pushing directly to $BRANCH (non-interactive, proceeding)."
  fi
fi

# Block if potential credentials detected
if git diff HEAD~1..HEAD | grep -E "(password|secret|api_key|token|private_key)\s*=" ; then
  echo "ERROR: Possible credential detected. Review before pushing."
  exit 1
fi

# Warn on large files >50 MB (BAM/VCF/h5ad/RDS)
LARGE=$(git diff --cached --name-only | xargs -I{} du -sm {} 2>/dev/null | awk '$1 > 50 {print $2}')
if [[ -n "$LARGE" ]]; then
  echo "WARNING: Large file(s) staged: $LARGE"
  if $IS_INTERACTIVE; then
    echo "Consider git-lfs. Continue? [y/N]"
    read -r answer < /dev/tty
    [[ "$answer" == "y" || "$answer" == "Y" ]] || exit 1
  else
    echo "WARNING: Large files detected (non-interactive, proceeding)."
  fi
fi

echo "Security check passed."
exit 0
```

> The `IS_INTERACTIVE` guard is required for HPC/non-interactive environments where `/dev/tty` is unavailable — without it the hook will block all pushes from Claude.

---

## 3. Claude Scanning Conversation History → Auto-Create Skills

### Concept
Claude Code can mine its own memory and session logs, surface high-frequency task patterns, and **write them as reusable Skills (SOPs)** — permanently available as slash commands.

### Step 1 — Ask Claude to Audit

```
Scan my conversation history and memory files at ~/.claude/projects/.
Identify the top 5 task types I run most frequently in my bioinformatics work.
For each, extract the optimal step sequence I've validated (ignore dead ends).
Output them as Skill file definitions I can save to ~/.claude/skills/.
```

### Step 2 — Example Auto-Generated Skills

**For Computational Biologist:**
```markdown
# Skill: new-omics-analysis

## Trigger
"Start analysis for <dataset>" / "set up <assay> pipeline"

## SOP
1. Create branch: `git checkout -b analysis/<dataset>-<assay>`
2. Copy template script from `templates/analysis_template.R`
3. Add metadata path and set seed at top of script
4. Confirm sample count matches metadata before proceeding
5. Commit scaffold: `analysis(<dataset>): scaffold <assay> pipeline`
6. Push and open draft PR

## Common Failure Modes
- Sample IDs mismatch metadata → check `colnames(counts) == metadata$sample_id`
- Missing conda env → `conda activate <env>` before running
```

**For Algorithm Developer:**
```markdown
# Skill: new-algorithm-module

## Trigger
"Add function <name>" / "implement <method>"

## SOP
1. Write function stub with NumPy docstring and type hints
2. Write unit test in `tests/test_<module>.py` with synthetic data
3. Implement function — keep ≤50 lines; extract helpers if needed
4. `pytest tests/test_<module>.py` — must pass before moving on
5. Update CHANGELOG.md under `[Unreleased]`
6. Commit: `feat(<module>): add <name> with tests`

## Common Failure Modes
- Test data too small → edge cases missed; use ≥3 samples, ≥10 features
- Magic numbers → define as module-level constants
```

### Step 3 — Register the Skill

```bash
# Via /update-config or directly in ~/.claude/settings.json
{
  "skills": [
    { "name": "new-omics-analysis",   "path": "~/.claude/skills/new-omics-analysis.md" },
    { "name": "new-algorithm-module", "path": "~/.claude/skills/new-algorithm-module.md" }
  ]
}
```

---

## 4. End-of-Session → SOP Distillation

### Concept
After any working session, ask Claude to strip out dead ends and crystallize the **clean optimal path** as a permanent Skill.

### Prompt to Run at Session End

```
Review everything we did in this conversation.
Identify the core repeatable task pattern.
Extract the optimal sequence — skip retries and wrong turns.
Write it as a Skill file for ~/.claude/skills/.
Include: trigger phrase, prerequisites, step-by-step SOP, and common failure modes.
Use terminology appropriate for computational biology / bioinformatics.
```

### Why This Matters for Bio Research
- Wet-lab → dry-lab handoffs follow the same preprocessing steps every time → **automate them**
- Journal revision rounds repeat the same figure-regeneration workflow → **one command**
- New lab member onboarding: share your `~/.claude/skills/` folder → instant knowledge transfer

---

## Summary

| Topic | Key Takeaway |
|---|---|
| `CLAUDE.md` Template A | Analysis-focused: reproducibility, paths, seeds, figure standards |
| `CLAUDE.md` Template B | Methods-focused: API contracts, tests, versioning, docs |
| Git hooks | Block bad pushes + warn on large bio files (BAM, h5ad, RDS) |
| Auto-push | SSH config once → Claude handles auth recovery automatically |
| Skill generation | Mine your session history → turn validated workflows into slash commands |

> **Rule of thumb:** If you've done it twice, turn it into a Skill.
> **Bio corollary:** If it's in your methods section, it should be a Skill.