---
description: Publish this project to GitHub — secret scan, push, README, About section, GitHub Pages, CI/CD, and Pages link
argument-hint: <owner/repo | https://github.com/owner/repo[.git]> [branch] [--private]
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
---

# /github123 — publish this project to GitHub

Target repository info from the user: `$ARGUMENTS`

You are publishing the current project to GitHub. Work through the steps below **in order**.
Step 1 (the secret scan) always runs **before** anything is pushed: nothing goes to the
internet until the scan is clean.

## Step 0 — Parse input and check prerequisites

1. Parse `$ARGUMENTS`:
   - Repository: accept `owner/repo`, `https://github.com/owner/repo`, `https://github.com/owner/repo.git`
     or `git@github.com:owner/repo.git`. Normalize to `OWNER` and `REPO`.
   - Optional branch name (default: the current branch, or `main` if the repo has no commits yet).
   - Optional `--private` flag (default visibility: public, because GitHub Pages on free plans needs a public repo).
   - If no repository was given, **stop and ask the user** for it. Do not guess.
2. Check the tooling:
   - `git --version`
   - `gh --version` and `gh auth status`. If `gh` is missing or not authenticated, use the GitHub MCP
     tools (`mcp__github__*`) where they cover a step; if neither is available, tell the user to run
     `gh auth login` and stop.
3. Inspect the project so later steps fit it: languages, build system (`package.json`, `CMakeLists.txt`,
   `pyproject.toml`, `requirements.txt`, `go.mod`, `Cargo.toml`, `pom.xml`, `build.gradle`, …),
   existing `README*`, existing `.github/workflows/`, existing `docs/` folder, and `.gitignore`.
4. Show the user a short plan (repo, branch, visibility, what will be created vs. edited) before continuing.

## Step 1 — Scan for sensitive information (blocking)

Scan **everything that would be pushed**: the working tree files tracked or about to be tracked
(`git ls-files` plus `git ls-files --others --exclude-standard`) **and the git history**
(`git log -p --all`), because a secret deleted in a later commit is still published with the history.

1. Run a dedicated scanner if one is available (prefer in this order), otherwise fall back to grep:
   - `gitleaks detect --source . --redact -v` (scans history) and `gitleaks dir . --redact`
   - `trufflehog git file://. --only-verified=false`
   - `detect-secrets scan`
2. Always also run a grep-based check (ripgrep / Grep tool) for at least:
   - Private keys: `-----BEGIN (RSA |EC |DSA |OPENSSH |PGP )?PRIVATE KEY-----`
   - AWS: `AKIA[0-9A-Z]{16}`, `aws_secret_access_key`
   - GitHub tokens: `gh[pousr]_[A-Za-z0-9]{36,}`, `github_pat_[A-Za-z0-9_]{20,}`
   - Google API keys: `AIza[0-9A-Za-z_-]{35}`
   - Slack: `xox[baprs]-[A-Za-z0-9-]+`, Slack webhook URLs
   - Stripe: `sk_live_[0-9a-zA-Z]{24,}`, `rk_live_`
   - OpenAI / Anthropic style keys: `sk-[A-Za-z0-9_-]{20,}`, `sk-ant-[A-Za-z0-9_-]+`
   - JWTs: `eyJ[A-Za-z0-9_-]+\.eyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+`
   - Connection strings with credentials: `(postgres|mysql|mongodb(\+srv)?|redis|amqp)://[^:\s]+:[^@\s]+@`
   - Generic assignments: `(password|passwd|pwd|secret|api[_-]?key|access[_-]?token|auth[_-]?token|client[_-]?secret)\s*[:=]\s*['"][^'"]{6,}['"]` (case-insensitive)
3. Look for files that should never be published: `.env`, `.env.*` (except `.env.example`),
   `*.pem`, `*.key`, `*.p12`, `*.pfx`, `id_rsa*`, `*.keystore`, `*.jks`, `credentials*.json`,
   `service-account*.json`, `.npmrc`/`.pypirc` containing tokens, `*.sqlite`/`*.db` dumps,
   `terraform.tfstate*`, `kubeconfig`, `.aws/`, `.ssh/`. Also flag personal data (real emails,
   phone numbers, internal hostnames/IPs) that look unintended.
4. Triage each hit: separate real secrets from false positives (test fixtures, documentation
   placeholders like `YOUR_API_KEY`, sample data). Report findings as a table:
   file, line, type, real/false-positive, action — **never print the full secret value**; show at most
   the first 4 characters followed by `…`.
5. If any real secret or sensitive file is found:
   - **Stop. Do not push.** Explain each finding and propose fixes:
     move the value to an environment variable / GitHub Actions secret, add the file to `.gitignore`,
     `git rm --cached <file>`, and provide a `.env.example` with placeholders.
   - If the secret is in git history, explain that it must be rewritten out of history
     (`git filter-repo` or BFG) **and rotated/revoked**, because it may already be compromised.
     Rewriting history is destructive — only do it with the user's explicit approval.
   - Re-run the scan after fixes. Continue only when it is clean or the user explicitly accepts
     each remaining item as a false positive.
6. Make sure `.gitignore` exists and covers secrets and build output for the project's stack
   (`.env`, `*.pem`, `*.key`, `node_modules/`, `build/`, `dist/`, `__pycache__/`, `.venv/`, IDE folders, …).

## Step 2 — Push the code to the GitHub repository

1. Check whether the repo exists: `gh repo view OWNER/REPO`.
   - If it does not exist, create it: `gh repo create OWNER/REPO --public|--private --source . --remote origin`
     (or `mcp__github__create_repository`). Confirm with the user before creating it.
2. Ensure the project is a git repo (`git init -b <branch>` if not) and that `origin` points to
   `https://github.com/OWNER/REPO.git`. If a different `origin` exists, ask before changing it
   (or add the new one under another remote name).
3. Commit any pending changes with a clear message.
4. Push: `git push -u origin <branch>`. On network failure retry up to 4 times with backoff
   (2s, 4s, 8s, 16s). If the push is rejected because the remote has commits, fetch and
   rebase/merge — **never force-push** without the user's explicit permission.

## Step 3 — Create or edit the README

1. If `README.md` exists, **edit** it: keep the existing content and the author's voice, fix
   outdated sections and add missing ones. If none exists, create one.
2. Base it on what the code actually does (read the entry points, build files and docs). Include:
   - Title, one-paragraph description, and badges (CI workflow status, GitHub Pages / website link, license)
   - Features
   - Getting started: prerequisites, installation, build and run commands that really work for this project
   - Usage examples
   - Project structure (short)
   - Testing
   - Contributing and License (link to `LICENSE` if present; do not invent a license — ask if there is none)
   - Link to the GitHub Pages site (added in Step 7)
3. Do not include any secret, internal URL or personal data in the README.
4. Commit and push.

## Step 4 — Create or edit the repository "About" section

Set the description, website and topics:

```bash
gh repo edit OWNER/REPO \
  --description "<concise one-line description, max ~350 chars>" \
  --add-topic <topic1> --add-topic <topic2> ...
```

- Topics: 5–10 lowercase, hyphenated topics drawn from the project's language, domain and frameworks.
- If an About description already exists, show it to the user and improve it rather than overwrite blindly.
- The website field is set in Step 7, once the Pages URL is known.

## Step 5 — Create a GitHub Pages site

1. Choose the source:
   - If the project already produces a static site (e.g. `docs/` with `index.html`, MkDocs, Docusaurus,
     Sphinx, Jekyll, Vite/React build), publish that.
   - Otherwise create a lightweight landing page in `docs/` (`docs/index.html` or `docs/index.md` with
     `docs/_config.yml`) built from the README: project name, description, features, install/usage,
     and a link back to the repository. Keep it responsive and dependency-free.
2. Deploy with GitHub Actions (preferred) using a workflow `.github/workflows/pages.yml` that uses
   `actions/configure-pages`, `actions/upload-pages-artifact` and `actions/deploy-pages`, with
   `permissions: { contents: read, pages: write, id-token: write }`, a `concurrency: pages` group,
   and triggers on `push` to the default branch plus `workflow_dispatch`. Add a build step first if the
   site needs building.
3. Enable Pages with the Actions build type:
   ```bash
   gh api -X POST repos/OWNER/REPO/pages -f build_type=workflow \
     || gh api -X PUT repos/OWNER/REPO/pages -f build_type=workflow
   ```
   If the repo is private on a plan that doesn't allow Pages, tell the user.
4. Commit, push, then wait for the deployment and get the URL:
   `gh run watch` (or `gh run list --workflow pages.yml`) and `gh api repos/OWNER/REPO/pages --jq .html_url`.
   The default URL is `https://OWNER.github.io/REPO/` (lowercase owner).

## Step 6 — Create CI/CD automation with GitHub Actions

1. Create `.github/workflows/ci.yml` (or extend the existing CI instead of duplicating it) that fits
   the detected stack, for example:
   - Node: `actions/setup-node` with cache, `npm ci`, lint, test, build
   - Python: `actions/setup-python` with a version matrix, install deps, lint (ruff/flake8), `pytest`
   - C/C++ (CMake): configure, build, `ctest`, optionally on an `ubuntu-latest`/`windows-latest`/`macos-latest` matrix
   - Java: `actions/setup-java`, Maven/Gradle build and test
   - Go / Rust: `go test ./...` / `cargo test`
2. Requirements for every workflow:
   - Triggers: `push` and `pull_request` on the default branch, plus `workflow_dispatch`
   - Least-privilege `permissions:` (default `contents: read`)
   - Pinned major versions of actions (e.g. `actions/checkout@v4`), dependency caching, `concurrency`
     to cancel superseded runs, and reasonable `timeout-minutes`
   - Secrets only through `${{ secrets.NAME }}` — never hard-coded. If the project needs secrets,
     tell the user which ones to add (`gh secret set NAME`) without asking them to paste values in chat.
3. Add a secret-scanning job to CI so future pushes are checked too (e.g. `gitleaks/gitleaks-action@v2`
   with `fetch-depth: 0`). Also suggest enabling GitHub secret scanning and push protection in the repo
   settings (`gh api -X PATCH repos/OWNER/REPO -f security_and_analysis[secret_scanning][status]=enabled
   -f security_and_analysis[secret_scanning_push_protection][status]=enabled` — available for public repos).
4. Optionally add `.github/dependabot.yml` for the detected package ecosystems and `github-actions`.
5. Validate the YAML (e.g. `actionlint` if available, or `python -c "import yaml,sys; yaml.safe_load(open(sys.argv[1]))" file`),
   commit, push, and check the run (`gh run list`, `gh run view --log-failed`). Fix failures caused by
   the workflow itself; report failures that come from the project's own code.

## Step 7 — Add the GitHub Pages link to the repository

1. Set the About website: `gh repo edit OWNER/REPO --homepage "https://OWNER.github.io/REPO/"`
   (use the real `html_url` from Step 5).
2. Make sure the README links to the Pages site (badge or "Website / Documentation" link) and the Pages
   site links back to the repo. Commit and push if anything changed.

## Step 8 — Final verification and report

1. Re-run the Step 1 secret scan against what is now on the remote branch (`git fetch && git log -p origin/<branch>`)
   and confirm it is clean.
2. Verify: `gh repo view OWNER/REPO --json description,homepageUrl,repositoryTopics,visibility`,
   Pages URL returns HTTP 200 (`curl -sI <pages-url>`), and the latest CI and Pages runs succeeded.
3. Report to the user with a concise checklist:
   - ✅/❌ Secret scan (with any items the user accepted as false positives)
   - ✅/❌ Code pushed (repo URL, branch, commit SHA)
   - ✅/❌ README created/updated
   - ✅/❌ About section (description, topics, website)
   - ✅/❌ GitHub Pages (URL)
   - ✅/❌ CI/CD workflows (files and latest run status)
   - Any follow-ups the user must do manually (secrets to add, keys to rotate, settings needing admin rights)

## Rules

- Never push before Step 1 is clean; never print secret values in full.
- Never force-push, delete branches, rewrite history, change visibility or overwrite an existing remote
  without explicit user approval.
- Prefer editing existing files (README, workflows, docs) over replacing them.
- If any step fails, explain why, continue with the steps that don't depend on it, and list it in the final report.
