# Build a Local AI Code Review Bot with Git Hooks and Ollama

Code review is a bottleneck. Every PR sits waiting for a senior dev to have 20 minutes. Meanwhile, obvious issues — unused imports, missing error handling, inconsistent naming — slip through because reviewers are focused on architecture, not housekeeping.

This tutorial builds a git hook that runs AI code review automatically on every commit and push. No CI pipeline required. No external API. The review runs locally in 5–30 seconds using Ollama, and your team never sees a push with obvious issues again.

## What We're Building

A two-stage AI review system:
1. **`pre-commit` hook** — reviews changed files before commit, blocks on critical issues
2. **`pre-push` hook** — comprehensive diff review against the target branch
3. A Python CLI (`review.py`) you can run manually on any file or diff

## Prerequisites

```bash
ollama pull llama3.2:3b          # fast reviewer, good for pre-commit
ollama pull qwen2.5-coder:7b    # deep review for pre-push
pip install gitpython rich click
```

## Project Structure

```
.git-hooks/
├── pre-commit          # shell script calling review.py
├── pre-push            # shell script calling review.py
└── review.py           # main reviewer
```

## The Core Reviewer

```python
# .git-hooks/review.py
#!/usr/bin/env python3
"""
AI Code Review via Ollama.
Usage:
  python review.py --staged          # review git staged changes
  python review.py --diff main       # review diff against branch
  python review.py --file foo.py     # review a specific file
"""
import subprocess
import sys
import click
import ollama
from rich.console import Console
from rich.panel import Panel
from rich.markdown import Markdown
from pathlib import Path

console = Console()

FAST_MODEL = "llama3.2:3b"       # pre-commit: speed over depth
DEEP_MODEL = "qwen2.5-coder:7b"  # pre-push: thorough review

BLOCKED_PATTERNS = [
    "CRITICAL",
    "security vulnerability",
    "SQL injection",
    "hardcoded password",
    "hardcoded secret",
    "command injection",
]

REVIEW_PROMPT = """You are a senior software engineer doing a code review.
Review the following code changes and respond in this exact format:

## Summary
One sentence describing what these changes do.

## Issues Found
List each issue as:
- [CRITICAL|WARNING|INFO] Description of issue, file:line if known

Use CRITICAL for: security vulnerabilities, data loss risks, broken logic, missing error handling on external calls.
Use WARNING for: code smells, missing tests, unclear naming, performance concerns.
Use INFO for: style suggestions, minor improvements.

## Verdict
APPROVE or BLOCK

BLOCK only if there are CRITICAL issues.

---
Changes to review:
{diff}
"""

FOCUSED_PROMPT = """You are reviewing this Python/JavaScript/TypeScript file for issues.
Be concise. List only real problems, not preferences.

Format:
- [CRITICAL|WARNING|INFO] issue description (line N if applicable)

File: {filename}
---
{content}
"""


def get_staged_diff() -> str:
    result = subprocess.run(
        ["git", "diff", "--cached", "--unified=5"],
        capture_output=True, text=True
    )
    return result.stdout


def get_branch_diff(base_branch: str = "main") -> str:
    result = subprocess.run(
        ["git", "diff", f"{base_branch}...HEAD", "--unified=5"],
        capture_output=True, text=True
    )
    return result.stdout


def get_staged_files() -> list[str]:
    result = subprocess.run(
        ["git", "diff", "--cached", "--name-only"],
        capture_output=True, text=True
    )
    return [f.strip() for f in result.stdout.splitlines() if f.strip()]


def is_reviewable(path: str) -> bool:
    SKIP_EXTENSIONS = {".lock", ".png", ".jpg", ".svg", ".ico", ".woff", ".woff2", ".ttf"}
    SKIP_DIRS = {"node_modules", ".git", "dist", "build", "__pycache__", ".venv"}
    p = Path(path)
    if p.suffix in SKIP_EXTENSIONS:
        return False
    for part in p.parts:
        if part in SKIP_DIRS:
            return False
    return True


def review_diff(diff: str, model: str = FAST_MODEL, max_chars: int = 8000) -> dict:
    """Run AI review on a diff. Returns {verdict, issues, summary, raw}."""
    if not diff.strip():
        return {"verdict": "APPROVE", "issues": [], "summary": "No changes to review", "raw": ""}

    # Truncate huge diffs
    if len(diff) > max_chars:
        diff = diff[:max_chars] + f"\n\n[... truncated at {max_chars} chars ...]"

    prompt = REVIEW_PROMPT.format(diff=diff)

    try:
        response = ollama.generate(
            model=model,
            prompt=prompt,
            options={"temperature": 0.1, "num_predict": 800},
        )
        raw = response["response"].strip()
    except Exception as e:
        console.print(f"[red]Ollama error: {e}[/red]")
        return {"verdict": "APPROVE", "issues": [], "summary": "Review skipped (model unavailable)", "raw": ""}

    # Parse verdict
    verdict = "APPROVE"
    if "BLOCK" in raw.upper().split("\n")[-5:]:
        verdict = "BLOCK"

    # Check for critical patterns
    raw_lower = raw.lower()
    for pattern in BLOCKED_PATTERNS:
        if pattern.lower() in raw_lower and "CRITICAL" in raw:
            verdict = "BLOCK"
            break

    # Extract issues
    issues = [
        line.strip()
        for line in raw.splitlines()
        if line.strip().startswith("- [")
    ]

    return {"verdict": verdict, "issues": issues, "summary": "", "raw": raw}


def review_file(filepath: str, model: str = FAST_MODEL) -> list[str]:
    """Review a single file. Returns list of issue strings."""
    path = Path(filepath)
    if not path.exists() or not is_reviewable(filepath):
        return []

    content = path.read_text(errors="replace")
    if len(content) > 6000:
        content = content[:6000] + "\n\n[... truncated ...]"

    prompt = FOCUSED_PROMPT.format(filename=filepath, content=content)

    try:
        response = ollama.generate(
            model=model,
            prompt=prompt,
            options={"temperature": 0.1, "num_predict": 400},
        )
        raw = response["response"].strip()
        return [
            line.strip()
            for line in raw.splitlines()
            if line.strip().startswith("- [")
        ]
    except Exception:
        return []


def print_review(result: dict, title: str = "AI Code Review"):
    raw = result.get("raw", "")
    verdict = result.get("verdict", "APPROVE")
    color = "green" if verdict == "APPROVE" else "red"

    console.print(Panel(
        Markdown(raw) if raw else "No output",
        title=f"[bold]{title}[/bold]",
        border_style=color,
    ))
    console.print(f"\n[bold {color}]Verdict: {verdict}[/bold {color}]")


@click.group()
def cli():
    pass


@cli.command()
@click.option("--model", default=FAST_MODEL, help="Ollama model to use")
@click.option("--block/--no-block", default=True, help="Exit 1 on BLOCK verdict")
def staged(model, block):
    """Review staged changes (pre-commit)."""
    diff = get_staged_diff()
    files = get_staged_files()
    reviewable = [f for f in files if is_reviewable(f)]

    if not reviewable:
        console.print("[dim]No reviewable files staged.[/dim]")
        sys.exit(0)

    console.print(f"[cyan]Reviewing {len(reviewable)} staged file(s)...[/cyan]")
    result = review_diff(diff, model=model)
    print_review(result, "Staged Changes Review")

    if block and result["verdict"] == "BLOCK":
        console.print("\n[bold red]Commit blocked. Fix CRITICAL issues above.[/bold red]")
        sys.exit(1)


@cli.command()
@click.argument("base", default="main")
@click.option("--model", default=DEEP_MODEL, help="Ollama model to use")
@click.option("--block/--no-block", default=True, help="Exit 1 on BLOCK verdict")
def diff(base, model, block):
    """Review diff against base branch (pre-push)."""
    d = get_branch_diff(base)
    if not d.strip():
        console.print("[dim]No diff against main. Nothing to review.[/dim]")
        sys.exit(0)

    console.print(f"[cyan]Deep review: diff against {base}...[/cyan]")
    result = review_diff(d, model=model, max_chars=12000)
    print_review(result, f"Branch Diff Review vs {base}")

    if block and result["verdict"] == "BLOCK":
        console.print("\n[bold red]Push blocked. Fix CRITICAL issues above.[/bold red]")
        sys.exit(1)


@cli.command("file")
@click.argument("filepath")
@click.option("--model", default=FAST_MODEL)
def file_review(filepath, model):
    """Review a single file."""
    issues = review_file(filepath, model=model)
    if not issues:
        console.print(f"[green]{filepath}: No issues found[/green]")
        return
    console.print(f"\n[bold]{filepath}:[/bold]")
    for issue in issues:
        color = "red" if "CRITICAL" in issue else "yellow" if "WARNING" in issue else "blue"
        console.print(f"  [{color}]{issue}[/{color}]")


if __name__ == "__main__":
    cli()
```

## Install the Hooks

```bash
# Install to your repo
cp .git-hooks/review.py /path/to/your/repo/.git-hooks/
chmod +x /path/to/your/repo/.git-hooks/review.py

# Create the pre-commit hook
cat > /path/to/your/repo/.git/hooks/pre-commit << 'EOF'
#!/bin/bash
python3 "$(git rev-parse --show-toplevel)/.git-hooks/review.py" staged
EOF
chmod +x /path/to/your/repo/.git/hooks/pre-commit

# Create the pre-push hook
cat > /path/to/your/repo/.git/hooks/pre-push << 'EOF'
#!/bin/bash
python3 "$(git rev-parse --show-toplevel)/.git-hooks/review.py" diff main
EOF
chmod +x /path/to/your/repo/.git/hooks/pre-push
```

Or use the installer:

```python
# install_hooks.py — run once per repo
import subprocess
import shutil
from pathlib import Path

def install(repo_path: str = "."):
    git_dir = Path(repo_path) / ".git"
    if not git_dir.exists():
        print("Not a git repo.")
        return

    hooks_dir = git_dir / "hooks"
    source_dir = Path(__file__).parent / ".git-hooks"

    # Copy reviewer
    shutil.copy(source_dir / "review.py", hooks_dir / "review.py")

    # pre-commit
    pre_commit = hooks_dir / "pre-commit"
    pre_commit.write_text(
        '#!/bin/bash\npython3 "$(git rev-parse --show-toplevel)/.git/hooks/review.py" staged\n'
    )
    pre_commit.chmod(0o755)

    # pre-push
    pre_push = hooks_dir / "pre-push"
    pre_push.write_text(
        '#!/bin/bash\npython3 "$(git rev-parse --show-toplevel)/.git/hooks/review.py" diff main\n'
    )
    pre_push.chmod(0o755)

    print(f"Hooks installed in {hooks_dir}")
    print("Run 'git commit' to test.")

install()
```

## What It Looks Like

When you `git commit`:

```
Reviewing 3 staged file(s)...
╭─────────────────────── Staged Changes Review ────────────────────────╮
│                                                                        │
│  ## Summary                                                            │
│  Adds user authentication endpoint with JWT token generation.          │
│                                                                        │
│  ## Issues Found                                                       │
│  - [CRITICAL] JWT secret is hardcoded as "mysecret" in auth.py:14     │
│  - [WARNING] No rate limiting on /login endpoint                       │
│  - [WARNING] Password compared without constant-time comparison        │
│  - [INFO] Consider extracting token expiry to config                   │
│                                                                        │
│  ## Verdict                                                            │
│  BLOCK                                                                 │
│                                                                        │
╰────────────────────────────────────────────────────────────────────────╯

Verdict: BLOCK

Commit blocked. Fix CRITICAL issues above.
```

The commit doesn't go through. Fix the hardcoded secret, re-stage, commit again:

```
Reviewing 3 staged file(s)...
...
Verdict: APPROVE

[main 3f8a2c1] Add user authentication endpoint
```

## Skipping the Hook

For work-in-progress commits where you're not ready for review:

```bash
git commit --no-verify -m "wip: auth skeleton"
```

Or set an env var in your hook to allow override:

```bash
# In .git/hooks/pre-commit
if [ "$SKIP_AI_REVIEW" = "1" ]; then
    exit 0
fi
```

```bash
SKIP_AI_REVIEW=1 git commit -m "hotfix: typo"
```

## Performance

On a modern laptop:
- `llama3.2:3b` reviews a 100-line diff in ~3 seconds
- `qwen2.5-coder:7b` does a 500-line diff in ~15 seconds

For large diffs (> 1000 lines), the reviewer truncates and focuses on the first 12KB — catching most issues while keeping the hook fast enough not to break flow.

## Extending It

**Custom rules** — add domain-specific checks before the AI runs:

```python
def check_custom_rules(diff: str) -> list[str]:
    issues = []
    if "console.log(" in diff:
        issues.append("[WARNING] console.log left in code")
    if "TODO" in diff:
        issues.append("[INFO] Unresolved TODO comment")
    if ".env" in diff and "password" in diff.lower():
        issues.append("[CRITICAL] Possible credential in .env change")
    return issues
```

**Team-wide installation** using pre-commit framework:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: local
    hooks:
      - id: ai-review
        name: AI Code Review
        entry: python .git-hooks/review.py staged --no-block
        language: python
        pass_filenames: false
        additional_dependencies: [ollama, rich, click, gitpython]
```

## Conclusion

The hook catches two categories of problems that humans consistently miss: security issues (hardcoded secrets, injection vulnerabilities) where familiarity blinds reviewers, and pattern violations that are obvious to a model trained on millions of codebases.

The local-only approach means no latency waiting for external APIs, no data leaving your network, and no per-call costs as your team's commit volume grows. Once Ollama is running, this costs nothing per review.

For teams already using pre-commit, drop `review.py` into your existing setup. For solo developers, the pre-commit hook alone will catch the kind of embarrassing issues that show up in PR comments — before the PR exists.
