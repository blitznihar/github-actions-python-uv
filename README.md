# Github Actions Python uv project
![Last Commit](https://img.shields.io/github/last-commit/blitznihar/github-actions-python-uv)
![License](https://img.shields.io/github/license/blitznihar/github-actions-python-uv?cacheSeconds=0)

This repository demonstrates a reusable GitHub Actions workflow for Python projects (UV-style example).

**Reusable GitHub Action**

- **Purpose:** Provide a reusable CI workflow that other repositories or workflows can call to run Python setup, lint, tests, and packaging steps.
- **Defined as:** a workflow that uses `workflow_call` inputs and can be invoked via `uses: owner/repo/.github/workflows/<workflow-file>.yml@ref`.

Example: Caller workflow that invokes the reusable workflow

```yaml
name: CI (caller)
on:
	push:
		branches: [main]

jobs:
	call-reusable:
		uses: blitznihar/github-actions-python-uv/.github/workflows/reusable.yml@main
		with:
			python-version: '3.11'
		secrets: inherit
```

Example: Reusable workflow (`.github/workflows/reusable.yml`)

```yaml
name: Reusable Python CI
on:
	workflow_call:
		inputs:
			python-version:
				description: 'Python version to use'
				required: false
				type: string
				default: '3.11'

jobs:
	test:
		runs-on: ubuntu-latest
		steps:
			- uses: actions/checkout@v4
			- name: Set up Python
				uses: actions/setup-python@v4
				with:
					python-version: ${{ inputs.python-version }}
			- name: Install dependencies
				run: |
					python -m pip install --upgrade pip
					pip install -r requirements.txt || true
			- name: Run tests
				run: |
					pytest -q

```

Usage notes

- Callers can pass inputs (like `python-version`) and inherit secrets with `secrets: inherit`.
- Pin the `uses:` reference to a tag or commit SHA in production to avoid unexpected breaking changes.
- Keep the reusable workflow focused and parameterized for easy reuse across projects.

If you want, I can add the `reusable.yml` workflow file to this repository and a simple test runner. Shall I create it?
<!-- INSERT: caller-workflow-steps -->

Using the central reusable workflows from each application repository

Follow these exact steps in each application repository (for example, `multiple-web-protocols`) so that each repo only contains tiny caller files that invoke the central workflows in this repository.

5A) Add `.python-version` to each repo

At the repo root create a file named `.python-version` with the desired Python version. This is your per-repo control point (no hard-coded Python in workflows):

```
3.11
```

5B) Replace local CI yaml with a caller workflow

Create (or replace) `.github/workflows/ci.yml` in each repo with the following caller workflow:

```yaml
name: CI

on:
	pull_request:
	push:
		branches: [ "main", "master" ]

jobs:
	ci:
		uses: blitznihar/github-actions-python-uv/.github/workflows/uv-ci.yml@main
		with:
			python-version-file: ".python-version"
			# OR use matrix:
			# python-versions-json: '["3.11","3.12","3.13"]'
			run-ruff: true
			run-semgrep: true
			run-tests: true
			run-pip-audit: true
			run-gitleaks: true
			run-trivy: true
			pip-audit-ignore: "CVE-2026-0994"
		secrets:
			CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
```

5C) Replace local release-please yaml with a caller

Create `.github/workflows/release-please.yml` in each repo:

```yaml
name: Release Please

on:
	push:
		branches: [ "main" ]

jobs:
	release:
		uses: blitznihar/github-actions-python-uv/.github/workflows/uv-release-please.yml@main
		with:
			package-name: "multiple-web-protocols"
```

5D) Replace local publish yaml with a caller (on tags)

Create `.github/workflows/publish-pypi.yml` in each repo:

```yaml
name: Publish to PyPI

on:
	push:
		tags:
			- "v*"

jobs:
	publish:
		uses: blitznihar/github-actions-python-uv/.github/workflows/uv-publish-pypi.yml@main
```

Step 6 — What to do with dependabot

Keep `.github/dependabot.yml` per repo. If you want to avoid manual copy/paste, choose one of:

- Make a template repository that includes `dependabot.yml` + minimal caller workflows.
- Use Copier (or another templating tool) to stamp/update config across repos.

Step 7 — Rollout checklist

For each target repo perform these steps:

- Add `.python-version`.
- Replace `.github/workflows/pylint.yml` (or your existing local CI) with the caller `ci.yml`.
- Replace `release-please.yml` with the caller version.
- Replace `publish-pypi.yml` with the caller version.
- Keep `dependabot.yml` (or copy it once from your template).

Notes

- Pin `uses:` references to a tag or commit SHA in production for stability.
- You can pass a JSON array of Python versions via `python-versions-json` to enable matrix runs from the central workflow.
- `python-version-file` points to the `.python-version` file in the target repo; the central workflow will read it.
