# Claude Code User Tips

This document compiles practical tips for using Claude Code, covering terminal interaction improvements, multi-AI collaborative code review, plan generation and automated PR review, SLURM job monitoring, and remote session control. These tips help you work more efficiently with Claude Code in daily development and research workflows.

---

## Overview

| # | Module | What It Does | Problem It Solves |
|---|--------|-------------|-------------------|
| 1 | [NO_FLICKER](#1-fullscreen-rendering--no_flicker-mode) | Terminal UI upgrade | Makes Claude Code feel like an IDE — mouse click, scroll, expand |
| 2 | [Codex Plugin](#2-codex-code-review-plugin) | Dual-AI code review | A second AI gatekeeps your code quality |
| 3 | [Humanize](#3-humanize--plan-generation--automated-pr-review) | From idea to PR | Automates plan generation, refinement, and PR review loops |
| 4 | [SLURM Monitor](#4-slurm-monitor-skill--auto-oom-detection--resubmission) | HPC auto-recovery | OOM jobs get resubmitted with more memory automatically |
| 5 | [Remote Control](#5-remote-control--control-claude-code-from-any-device) | Cross-device control | Continue your session from phone, tablet, or another browser |

---

## 1. Fullscreen Rendering / NO_FLICKER Mode

### Purpose

Enable **fullscreen rendering** (officially a *research preview*) in Claude Code. Once enabled, Claude Code handles mouse events inside the terminal, giving you:

- Click the input box to move the cursor to any position
- Click collapsed tool outputs to expand them
- Click links or file paths to open them
- Drag to select text
- Scroll with the mouse wheel

This eliminates the need to press arrow keys or use `Ctrl+O` to open an external editor. The input box stays pinned at the bottom instead of flickering upward with each output.

**Requirement**: Claude Code **v2.1.89+**.

### Step-by-Step Setup

#### 1. Check version and update if needed

```bash
claude --version
claude update
```

#### 2. Try it once (temporary)

```bash
CLAUDE_CODE_NO_FLICKER=1 claude
```

If it works, the input box stays fixed at the bottom while Claude is working.

#### 3. Make it permanent

**Option A** — Shell config (`~/.bashrc` or `~/.zshrc`):

```bash
echo 'export CLAUDE_CODE_NO_FLICKER=1' >> ~/.bashrc
source ~/.bashrc
```

**Option B** — Claude Code `settings.json`:

Any environment variable can be placed under the `env` key. User-level file is `~/.claude/settings.json`; project-level file is `.claude/settings.json`.

```json
{
  "env": {
    "CLAUDE_CODE_NO_FLICKER": "1"
  }
}
```

### Troubleshooting: Mouse Wheel Not Working

If the mouse wheel doesn't respond after enabling, check your terminal environment first.

**For tmux users** — add to `~/.tmux.conf`:

```bash
set -g mouse on
```

> **Note**: `tmux -CC` (control mode) is **not compatible** with fullscreen rendering.

**For iTerm2 users** — make sure **Enable mouse reporting** is turned on in iTerm2 preferences.

**Scroll speed too slow?** Adjust with:

```bash
export CLAUDE_CODE_SCROLL_SPEED=3
```

### Keep Native Terminal Text Selection

If you prefer the terminal's native drag-to-select-and-copy behavior over Claude Code's mouse handling, you can disable Claude Code's mouse capture while keeping the flicker-free rendering:

```bash
CLAUDE_CODE_NO_FLICKER=1 CLAUDE_CODE_DISABLE_MOUSE=1 claude
```

This keeps the no-flicker rendering but disables click-to-position-cursor, click-to-expand, click links, and scroll wheel inside Claude Code.

### Known Issues

This feature is labeled **research preview** by Anthropic. Some known issues reported on the official GitHub:

- On **Windows Terminal**: after enabling `NO_FLICKER`, the scroll wheel may work but clicking the input box may **not** move the cursor to the click position.
- **Fast scrolling** can sometimes cause escape-sequence text to appear in the input box.

These are compatibility issues on Anthropic's side, not configuration errors.

### References

- Claude Code Official Documentation: <https://docs.anthropic.com/en/docs/claude-code>
- Claude Code GitHub Repository: <https://github.com/anthropics/claude-code>
- Claude Code Changelog: <https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md>

---

## 2. Codex Code Review Plugin

### Purpose

Integrate OpenAI's Codex capabilities into your Claude Code workflow, enabling a dual-AI collaboration model: Claude Code writes the code, Codex reviews and gatekeeps. Two top-tier AIs reviewing each other's work leads to higher code quality.

### Installation

**Step 1**: Install the Codex CLI (if not already installed)

```bash
npm install -g @openai/codex
```

**Step 2**: Install the plugin in Claude Code

```
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
/reload-plugins
/codex:setup
```

**Step 3**: Restart Claude Code for the plugin to take effect.

### Core Commands

| Command | Description |
|---------|-------------|
| `/codex:review` | Standard code review — Codex reviews your code in read-only mode, providing optimization suggestions and bug reports |
| `/codex:adversarial-review` | Adversarial review — Codex aggressively challenges your code: logic correctness, edge cases, performance issues, etc. |
| `/codex:rescue` | Delegate task — Hand off a task that Claude Code struggles with to Codex for background execution |
| `/codex:status` | Check background task status |
| `/codex:result` | Retrieve background task results |
| `/codex:cancel` | Cancel a background task |

### Examples

```bash
# Claude Code just finished writing code — let Codex review it
/codex:review

# Want a stricter review (edge cases, performance, security)
/codex:adversarial-review

# Claude Code is stuck — hand the task off to Codex
/codex:rescue
/codex:status          # check progress
/codex:result          # get results
```

### Configuration

You can configure Codex's default reasoning level and model in `.codex/config.toml`. For deeper reviews, increase the reasoning effort setting.

### Notes

- **Do NOT enable auto-review**: While the plugin supports automatic review before every code change, this causes Claude and Codex to enter a long loop, burning through your token quota rapidly. Only invoke reviews manually when needed.
- **Use adversarial review for critical code**: `/codex:adversarial-review` is especially valuable when writing business-critical logic or security-sensitive code.
- **Restart after installation**: You must restart Claude Code after installing the plugin, otherwise the new commands may not be recognized.

### References

- Codex Plugin GitHub: <https://github.com/openai/codex-plugin-cc>
- OpenAI Codex CLI GitHub: <https://github.com/openai/codex>
- OpenAI Codex Official Page: <https://openai.com/index/introducing-codex/>

---

## 3. Humanize — Plan Generation & Automated PR Review

### Purpose

A Claude Code plugin that turns a draft document into a structured implementation plan, refines it with a QA ledger, and then drives an automated PR review loop. It bridges the gap between "rough idea" and "merged PR" by combining plan generation, Codex-powered review, and bot-monitored PR workflows.

### How It Works

1. **Generate a plan** — Feed a draft document (design doc, feature spec, bug description, etc.) to `humanize:gen-plan`. It produces a structured, step-by-step implementation plan.
2. **Refine the plan** — Run `humanize:refine-plan` to annotate, clean up, and generate a QA ledger that tracks acceptance criteria.
3. **Execute with review loops** — Start a PR loop (`humanize:start-pr-loop`) or a Codex review loop (`humanize:start-rlcr-loop`) that automatically monitors and reviews code changes.
4. **Ask Codex** — At any point, use `humanize:ask-codex` to consult Codex as an independent expert on a specific question.

### Installation

```bash
# Add the humania marketplace
/plugin marketplace add humania-org/humanize

# For experimental features, use the dev branch
/plugin marketplace add humania-org/humanize#dev

# Install the humanize plugin
/plugin install humanize@humania

# Reload plugins to activate
/reload-plugins
```

### Core Skills

| Skill | Description |
|-------|-------------|
| `/humanize:gen-plan` | Generate an implementation plan from a draft document |
| `/humanize:refine-plan` | Refine an annotated plan and generate a QA ledger |
| `/humanize:start-pr-loop` | Start a PR review loop with bot monitoring |
| `/humanize:start-rlcr-loop` | Start an iterative review loop with Codex |
| `/humanize:ask-codex` | Send a question or task to Codex and get an independent expert response |

### Examples

```bash
# You have a draft design doc — generate a structured implementation plan
/humanize:gen-plan

# Refine the plan: clean up annotations, produce a QA ledger
/humanize:refine-plan

# Start an automated PR review loop (bot watches for changes and reviews)
/humanize:start-pr-loop

# Start a Codex-powered iterative review loop
/humanize:start-rlcr-loop

# Ask Codex a one-off question as an independent expert
/humanize:ask-codex "Is this retry logic safe under concurrent access?"
```

### References

- Humanize GitHub Repository: <https://github.com/humania-org/humanize>

### Notes

- Requires the `humania-org/humanize` marketplace to be added first
- After installation, always run `/reload-plugins` to activate new skills
- Check the official repository for the latest features and version updates

---

## 4. SLURM Monitor Skill — Auto OOM Detection & Resubmission

### Purpose

When running large batches of compute jobs on HPC clusters via SLURM, Out of Memory (OOM) failures are common. This Claude Code Skill auto-generates a `monitor.sh` script that:

- Automatically detects OOM-killed jobs
- Resubmits them with increased memory
- Generates a final summary report

### Skill File Location

```
skills/slurm-monitor/SKILL.md
```

### Supported Job Modes

| Mode | Trigger | Description |
|------|---------|-------------|
| **Batch mode** | Input is a directory containing `batch_*.sh` scripts | Monitors each batch script individually |
| **Array mode** | Input is a `.sh` file with `#SBATCH --array=...` | Monitors each task in the array job individually |

Auto-detection: file → Array mode, directory → Batch mode.

### Memory Retry Strategies

| Strategy | Condition | Behavior |
|----------|-----------|----------|
| **Iterative increment** (default) | `mem_increment > 0` | Increase memory by a fixed amount on each OOM, up to `max_mem`. E.g., 20G → 30G → 40G → ... |
| **Single retry** | `mem_increment = 0` | Jump directly from `initial_mem` to `max_mem` on OOM |

### How to Use

Invoke the skill in Claude Code:

```
/slurm-monitor <batch_dir_or_script> <job_prefix> [initial_mem] [mem_increment] [max_mem] [check_interval] [output_path]
```

**Parameters:**

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `batch_dir_or_script` | Yes | — | Directory of batch scripts or path to an array job script |
| `job_prefix` | Yes | — | The `--job-name` prefix used in SBATCH scripts |
| `initial_mem` | No | 50G | Memory configured in the original scripts |
| `mem_increment` | No | 50G | Memory to add on each OOM retry (set to 0 for single retry) |
| `max_mem` | No | 200G | Maximum memory cap; stops retrying beyond this |
| `check_interval` | No | 1800s | Monitoring check interval in seconds |
| `output_path` | No | Auto-generated | Output path for `monitor.sh` |

### Examples

**Example 1**: Monitor a batch directory, starting at 20G, incrementing by 10G, max 60G

```
/slurm-monitor /path/to/batch_scripts my_job 20G 10G 60G
```

**Example 2**: Monitor an array job script with default parameters

```
/slurm-monitor /path/to/submit.sh my_array_job
```

**Example 3**: Single retry mode (jump directly to max memory on OOM)

```
/slurm-monitor /path/to/batch_scripts my_job 50G 0 200G
```

After the script is generated, submit the monitor job:

```bash
sbatch /path/to/logs/monitor.sh
```

### Output

When monitoring completes, a summary report (`monitor_summary.txt`) is generated, containing:

- Total tasks, first-run successes, retried tasks
- Retry successes broken down by memory level
- Tasks that hit max memory and still OOM'd
- Non-OOM failures with task IDs listed
- Overall completion rate

### Notes

- **Duplicate prevention**: On startup, the monitor checks if another monitor with the same `job_prefix` is already running; if so, it exits immediately.
- **State persistence**: Array mode uses a state file so the monitor can be safely restarted without losing track of progress.
- **Idempotent**: Re-running the monitor is safe — it checks state before taking any action.
- **Stuck detection**: If 3 consecutive check cycles pass with no running jobs and no resubmissions, the monitor exits automatically to prevent infinite loops.
- **Auto-detected log directory**: The log directory is parsed from the `--output` line in SBATCH scripts — no manual specification needed.

### References

- Skill definition: `skills/slurm-monitor/SKILL.md`
- Claude Code Custom Skills Documentation: <https://docs.anthropic.com/en/docs/claude-code/skills>
- SLURM Official Documentation: <https://slurm.schedmd.com/documentation.html>

---

## 5. Remote Control — Control Claude Code from Any Device

### Purpose

Control a running Claude Code session from your phone, tablet, or any web browser. Start a task at your desk, then continue steering it from another device — while keeping full access to your local filesystem, MCP servers, tools, and project configuration. Everything still executes on your machine; no code or data moves to the cloud.

**Requirement**: Claude Code **v2.1.51+**, with a **Pro, Max, Team, or Enterprise** subscription (API keys not supported).

### How to Start

**Option 1** — Enable Remote Control in an existing session:

```
/remote-control
```

Or with a custom session name:

```
/remote-control My Project
```

**Option 2** — Start a new session with Remote Control enabled:

```bash
claude --remote-control
claude --remote-control "My Project"
```

**Option 3** — Start a dedicated Remote Control server:

```bash
claude remote-control
```

### How to Connect from Another Device

Once Remote Control is active, you have three ways to connect:

| Method | How |
|--------|-----|
| **Browser** | Open the session URL displayed in your terminal directly in any browser (goes to `claude.ai/code`) |
| **QR Code** | Scan the QR code shown in the terminal with your phone (press spacebar in server mode to show it) |
| **claude.ai/code** | Find the session by name in the session list — active sessions show a green status dot |

The conversation stays in sync across all connected devices. You can send messages from your terminal, browser, and phone interchangeably.

### Examples

**Example 1**: Start working at your desk, continue from your phone

```bash
# At your desk terminal
claude --remote-control "Training Pipeline"

# Start a long-running task
> Help me debug the data preprocessing pipeline

# Walk away — open claude.ai/code on your phone, find "Training Pipeline", continue the conversation
```

**Example 2**: Enable Remote Control mid-session

```
# Already in a Claude Code session, realize you need to step away
/remote-control

# Scan the QR code with your phone and keep going
```

**Example 3**: Run a dedicated server for multiple sessions

```bash
claude remote-control
# Use --spawn for multiple concurrent sessions
```

### Key Features

- **Local execution**: All commands run on your machine — your local tools, MCP servers, and files stay available
- **No inbound ports**: Only makes outbound HTTPS requests; no ports are opened on your machine
- **Auto-reconnect**: If your network drops briefly, the session reconnects automatically
- **Sync across devices**: Messages sent from any device appear on all connected surfaces

### Notes

- **Terminal must stay open**: If you close the terminal, the session ends
- **One remote session per process**: Use server mode with `--spawn` for multiple concurrent sessions
- **Network timeout**: Extended outages over ~10 minutes will timeout the session
- **Team/Enterprise**: Admin must enable the Remote Control toggle in admin settings
- **Authentication**: You must be logged in via `/login` with a claude.ai account (not an API key)
- **Ultraplan conflict**: Starting an ultraplan session disconnects any active Remote Control session

### References

- Claude Code Remote Control Documentation: <https://docs.anthropic.com/en/docs/claude-code/remote-control>
- Claude Code Official Documentation: <https://docs.anthropic.com/en/docs/claude-code>
