# Repository Guidelines

## Project Structure & Module Organization

This workspace contains two kinds of material:

- `kakao_clone_coding_lecture/`: the notebook-based KanaMate agentic AI course. Weekly notebooks live in `notebook/`, supporting images in `notebook/imgs/`, and lesson summaries in `docs/`.
- `학습일지/`: personal weekly learning logs and the submission guide.

The course directory is a nested Git repository and owns its Python environment files (`pyproject.toml`, `uv.lock`, `.python-version`). Run development commands from that directory. Keep lesson-specific edits in the matching `weekNN` notebook or document; do not mix learning-log content into course source files.

## Build, Test, and Development Commands

```bash
cd kakao_clone_coding_lecture
bash scripts/setup_uv_env.sh
uv sync --locked
uv run jupyter lab
```

The setup script prepares the expected Python 3.10 environment. `uv sync --locked` installs exactly the dependencies recorded in `uv.lock`, and Jupyter Lab opens the weekly exercises locally. There is no separate build step.

Before submitting notebook changes, restart the kernel and run all cells from top to bottom. For lightweight repository checks, use:

```bash
git diff --check
git status --short
```

## Coding Style & Naming Conventions

Use four spaces for Python indentation and follow existing PEP 8-style naming: `snake_case` for functions and variables, `PascalCase` for Pydantic models, and `UPPER_CASE` for constants. Add type annotations to new Python interfaces. Preserve the established Korean lesson filenames, such as `2주차_자연어를_구조화된_요청으로_만든다.ipynb`, and use `weekNN` prefixes for logs and lesson documents. Avoid unrelated notebook output or formatting churn.

## Testing Guidelines

No automated test suite or coverage threshold is configured. Treat a clean full-notebook execution as the primary verification. Exercise representative inputs, including expected, ambiguous, and missing-value cases, and confirm structured outputs match their Pydantic schema. Never commit API tokens or live credentials in notebook cells or outputs.

## Commit & Pull Request Guidelines

Recent history uses short, scoped subjects such as `week2: 구조화 요청 agent 구현`, `docs: ...`, and `chore: ...`. Follow `<scope>: <imperative summary>` and keep each commit focused. Pull requests should state the week and learning objective, summarize changed files, list verification performed, and link the relevant issue. Include screenshots only when rendered notebook or UI behavior is important to review.

## Security & Configuration

Copy `.env.example` to `.env` for local secrets. Keep `.env`, virtual environments, caches, and generated data out of commits; update `.env.example` only with placeholder values.
