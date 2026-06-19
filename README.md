# auto-pr

Shared CI/CD and code review pipeline for ThaliosAI repos. Every PR automatically runs Ruff, Pytest, and a local LLM review via PR-Agent (Qwen3 30B).

## How it works

```
[ Developer Opens PR ]
        │
        ▼
[ Ruff ] ──────────────> Fails on style errors or syntax issues
        │ Pass
        ▼
[ Pytest + Coverage ] ──> Fails if tests break or coverage drops below threshold
        │ Pass
        ▼
[ PR-Agent (Qwen3 30B) ] > Inline comments on logic, edge cases, security
        │
        ▼
[ Human Reviewer ] ──────> Focuses on architectural decisions only
```

The Ruff and Pytest jobs run on GitHub-hosted runners. The PR-Agent job runs on the ThaliosAI self-hosted runner (`thalios-workstation-1`) which has local access to Ollama.

## Adding auto-pr to a new repo

### 1. Add the CI workflow

```bash
mkdir -p .github/workflows
cp /path/to/auto-pr/caller-example.yml .github/workflows/pr-review.yml
sed -i "s/YOUR_ORG/ThaliosAI/g" .github/workflows/pr-review.yml
```

### 2. Add pre-commit hooks

```bash
cp /path/to/auto-pr/.pre-commit-config.yaml .
```

### 3. Add ruff config to `pyproject.toml`

```toml
[tool.ruff.lint]
ignore = ["E741"]  # adjust per repo as needed
```

### 4. Commit and push

```bash
git add .github/workflows/pr-review.yml .pre-commit-config.yaml pyproject.toml
git commit -m "Add auto-pr workflow and pre-commit hooks"
git push
```

The CI pipeline will trigger on the next PR opened against the default branch.

## Developer setup (per person, once after cloning)

Every developer needs to install and activate the pre-commit hooks locally:

```bash
pipx install pre-commit
pre-commit install
```

After this, Ruff runs automatically on every `git commit`. If it modifies files, stage and commit again:

```bash
git add .
git commit -m "your message"
```

To scan the entire repo for pre-existing issues (useful after first installing hooks):

```bash
pre-commit run --all-files
```

## Customising per repo

The workflow accepts optional inputs to override defaults. Add a `with:` block to the caller workflow:

```yaml
jobs:
  review:
    uses: ThaliosAI/auto-pr/.github/workflows/pr-review.yml@master
    secrets: inherit
    with:
      python-version: "3.12"
      coverage-threshold: 85
      install-cmd: "pip install -r requirements-dev.txt"
      ollama-model: "qwen3:30b"
```

| Input | Default | Description |
|---|---|---|
| `python-version` | `3.11` | Python version for lint and test jobs |
| `coverage-threshold` | `80` | Minimum test coverage % |
| `install-cmd` | auto-detected | Command to install project dependencies |
| `test-cmd` | `pytest -q` | Pytest invocation |
| `ollama-model` | `qwen3:30b` | Ollama model for PR-Agent |
| `ollama-host` | `http://localhost:11434` | Ollama API URL |
| `ollama-num-ctx` | `32768` | Context window size (tokens) |

## Infrastructure

- **Self-hosted runner**: `thalios-workstation-1` — registered at the org level, tagged `self-hosted, llm`
- **Ollama**: running locally on the workstation, serving `qwen3:30b`
- **PR-Agent**: runs via `docker run codiumai/pr-agent:latest` with `--network host`
