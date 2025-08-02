---

# ✅ Dev Workflow – Production-Level Git + CI/CD Practices (Explained Fully)

---

## 🧩 1. Git Branching Strategy (Collaboration & Code Isolation)

### 🟢 Common Branches:

| Branch            | Purpose                                   |
| ----------------- | ----------------------------------------- |
| `main` / `master` | Stable, production-ready code             |
| `dev`             | Ongoing development integration           |
| `feature/*`       | New features (e.g., `feature/auth-flow`)  |
| `bugfix/*`        | Fix specific bugs (`bugfix/login-error`)  |
| `hotfix/*`        | Urgent production fixes                   |
| `release/*`       | Prep branch for major version deployments |

### 🔁 Example Flow:

```bash
git checkout -b feature/signup-form
# work...
git add .
git commit -m "feat(auth): add signup form"
git push origin feature/signup-form
```

---

## 📝 2. PRs (Pull Requests) – Code Review Workflow

### 🔹 Why:

* Enable **team review**, comments, approvals
* Prevent broken code from merging into `main`

### 🔹 Best Practices:

| Practice                   | Why it matters              |
| -------------------------- | --------------------------- |
| Small PRs (max \~300 LOC)  | Easier to review            |
| PR Title = Commit Message  | Consistency                 |
| Use draft PRs for WIP      | Share without merging       |
| Assign reviewers           | Accountability              |
| Resolve conversations      | Clean history               |
| Rebase/squash before merge | Prevent messy merge commits |

### ✅ PR Template Example (`.github/pull_request_template.md`)

```md
## What this PR does:
- [x] Adds auth middleware
- [x] Protects /dashboard route

## Checklist:
- [x] Tested locally
- [x] No console.logs
- [x] Follows naming convention

## Issue Reference:
Fixes #12
```

---

## 🧾 3. Commit Conventions – Semantic, Readable History

Use **Conventional Commits** format:

```bash
<type>(scope): <description>
```

| Type       | Use For                      |
| ---------- | ---------------------------- |
| `feat`     | New feature                  |
| `fix`      | Bug fix                      |
| `docs`     | Documentation only           |
| `style`    | Formatting, no logic change  |
| `refactor` | Code change w/o bug/feature  |
| `test`     | Adding tests                 |
| `chore`    | Tooling, CI changes, updates |

### 🔹 Examples:

* `feat(auth): implement JWT login`
* `fix(api): correct status code for 403`
* `chore(deps): update mongoose to v7.1.0`

### 🔧 Optional: Enforce with [commitlint](https://commitlint.js.org/)

---

## 🗂️ 4. .gitignore – Keep Secrets & Junk Out of Git

### 🔹 Purpose:

Prevent committing:

* Secrets
* Binaries
* Temporary files
* Dependencies

### ✅ Example `.gitignore`:

```
node_modules/
.env
dist/
coverage/
*.log
.DS_Store
```

### 🔒 Secrets Management:

* Store secrets in `.env`
* Use `dotenv`, AWS Secrets Manager, or Vault
* Never commit `.env` or `secrets.json`

---

## ⚙️ 5. GitHub Actions CI – Automate Lint → Test → Build → Deploy

### 📦 What is CI/CD?

**CI (Continuous Integration)** = Automatically run tests & checks on every push/PR
**CD (Continuous Deployment)** = Automatically deploy after passing checks

---

## 🛠 6. Setting Up GitHub Actions (Step-by-Step)

### ✅ 1. Create Workflow File

> File: `.github/workflows/ci.yml`

```yaml
name: Node.js CI

on:
  push:
    branches: [main, dev]
  pull_request:
    branches: [main, dev]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: 📥 Checkout code
      uses: actions/checkout@v3

    - name: 🔧 Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'

    - name: 📦 Install dependencies
      run: npm ci

    - name: 🔍 Lint code
      run: npm run lint

    - name: 🧪 Run tests
      run: npm test

    - name: 🚀 Build project
      run: npm run build
```

---

## 🧪 7. Lint → Test → Build – Explained

### 🔹 Lint

* Catches code quality issues
* Run with ESLint:

```json
"scripts": {
  "lint": "eslint ."
}
```

### 🔹 Test

* Run unit/integration tests

```json
"scripts": {
  "test": "jest"
}
```

### 🔹 Build

* Compile/prepare code for production

```json
"scripts": {
  "build": "tsc" // for TypeScript
}
```

---

## 🚀 8. Deployment (Optional CD Step)

### ✈️ Deploy with GitHub Actions (to Vercel/Heroku/AWS):

Example: Deploy to Vercel

```yaml
    - name: 🚀 Deploy to Vercel
      run: vercel --token=${{ secrets.VERCEL_TOKEN }} --prod
```

Example: Deploy Lambda via Serverless CLI

```yaml
    - name: Deploy to AWS
      run: npx serverless deploy
```

---

## 📊 9. Extras for Real-World CI/CD

| Feature                   | How                                       |
| ------------------------- | ----------------------------------------- |
| **Code coverage reports** | Use `nyc` or `c8` with `jest`             |
| **Slack notifications**   | `slackapi/slack-github-action`            |
| **Test matrix**           | Run same job on multiple Node.js versions |
| **Secrets**               | Set in GitHub → Repo Settings → Secrets   |
| **Fail fast**             | Stop pipeline if lint/test fails          |

---

## ✅ Final Dev Workflow Checklist

* [x] Use Git branching (feature/bugfix/hotfix)
* [x] Commit using Conventional Commit format
* [x] Open PR with clear description and linked issues
* [x] CI runs lint → test → build on every PR/push
* [x] Secrets in `.env`, excluded via `.gitignore`
* [x] Deployment triggered via GitHub Actions (optional)
* [x] Peer reviews and squash merges into `main`

---
