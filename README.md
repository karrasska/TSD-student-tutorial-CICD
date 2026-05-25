# 📦 Setup

Please **fork** the repo. You can work locally using a code editor, or open it in GitHub Codespaces.

---

# 🟢 Task 1 (5 pts) - CI Quality Gate: Tests + Lint + Formatting

## 🎯 Goal
Set up a **GitHub Actions CI workflow** that runs automatically for every **Push** and **Pull Request** and blocks merging if:
- unit tests fail
- linting fails
- formatting check fails

You should get a ✅/❌ status check on the PR.

---

## ✅ Requirements
1. Create a workflow file:  
   **`.github/workflows/ci.yml`**

2. The workflow must run on:
- `pull_request` events (PR opened / updated)

3. The workflow must run **all of the following** checks:
- **Tests**: `pytest` (or `python manage.py test`)
- **Lint**: `ruff check .`
- **Formatting**: `black --check .`

4. If any check fails, the workflow must fail.

---

## 💡 Hints (Fill in the blanks!)

To build your workflow, you need to combine standard GitHub Actions. You will need to look up the exact names of the actions (like checkout and setup-python) on the GitHub Marketplace.

```yaml
name: CI Pipeline
on: [pull_request]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Check out code
        uses: # TODO: Find the standard checkout action (v4)

      - name: Set up Python
        uses: # TODO: Find the standard setup-python action (v5)
        with:
          # TODO: Specify python-version (e.g., '3.10') and enable pip caching

      - name: Install dependencies
        run: pip install -r requirements-dev.txt

      - name: Run Formatting Check
        run: # TODO: Write the command to run black in check mode

      - name: Run Linter
        run: # TODO: Write the command to run ruff

      - name: Run Tests
        run: # TODO: Write the command to run pytest
```

---

## 📦 How to show your work?
- Change something in the code on a new branch and push it.
- Open a Pull Request containing your workflow file.
- Ensure the PR shows the CI checks and they are ✅ green.
- Break the tests. Push the changes and see that the checks are ❌.

---

# 🟡 Task 2 (10 pts) - DevSecOps + PR Automation (CodeQL + Dependabot + PR Comment)

## 🎯 Goal
Enhance your repository with **security automation** and **PR feedback** by implementing:
- **CodeQL code scanning** (results visible in the repository **Security** tab)
- **Dependabot** dependency update configuration
- An **automatic PR comment** posted when CI is ✅ green (tests + lint + formatting passed)

---

## ✅ Requirements (and Guides)

### (A) CodeQL (Code Scanning → Security tab)

**💡 How to do it:** You don't need to write this from scratch! 
1. Go to your repository on GitHub.
2. Click the **Security** tab -> **Code scanning** -> **Configure scanning tool**.
3. Choose **CodeQL Analysis** (Advanced Setup). 
4. GitHub will generate `codeql.yml` for you. Just commit it to your repository!

---

### (B) Dependabot (automatic dependency update PRs)

**💡 How to do it:** Dependabot doesn't use standard workflows. It uses a specific config file.
1. Create `.github/dependabot.yml`.
2. Use this template and consult the GitHub Dependabot documentation to fill in the blanks:

```yaml
version: 2
updates:
  - package-ecosystem: # TODO: What is the ecosystem name for Python pip?
    directory: # TODO: Where are your requirements files located? (usually "/")
    schedule:
      interval: # TODO: Set this to weekly
```

---

### (C) Automatic PR comment when CI succeeds

**💡 How to do it:** 1. Open your `ci.yml` from Task 1.
2. Add a **second job** right below your first one. 
3. Use `actions/github-script` to interact with the GitHub API. 

```yaml
  pr-comment:
    needs: # TODO: Which job must finish successfully before this runs?
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write # Required to post a comment
    steps:
      - name: Comment on PR
        uses: actions/github-script@v7
        with:
          script: |
            // TODO: Use github.rest.issues.createComment to post a message.
            // You will need to pass an object with: issue_number, owner, repo, and body.
            // Check out the actions/github-script docs for an exact example!
```

---

## 📦 How to show your work?
- Open a PR with the new files.
- Ensure:
  - CodeQL workflow runs successfully
  - Dependabot config is present
  - PR gets a ✅ success comment after green CI

---

# 🔴 Task 3 (15 pts) - Build → Scan → Push Docker image (GitHub Secrets required)

## 🎯 Goal
Create a GitHub Actions workflow that works like a **delivery pipeline**:

1) **Build** the Docker image from this repository  
2) **Scan** the image with a **non-GitHub** container security scanner (Trivy)  
3) **Push** the image to **Docker Hub** 4) Authenticate using **GitHub Secrets** (no credentials in code)

---

## ✅ Requirements (and Guides)

### (A) Docker Hub auth via GitHub Secrets
1. Go to your repository **Settings → Secrets and variables → Actions**.
2. Add `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` (token provided during class).

### (B) Setup the Workflow Skeleton
1. Create `.github/workflows/docker.yml`.
2. Trigger it **only** on `push` to `main` (never `pull_request` for deployments!).

### (C, D, E) The Build, Scan, and Push Job

**💡 The Workflow Steps:**
You will need to search the GitHub Marketplace for the official Docker actions (`login-action`, `build-push-action`) and the Aqua Security Trivy action.

```yaml
    steps:
      - name: Check out code
        uses: # Use the same checkout action as in Task 1
        
      - name: Get Short SHA
        run: echo "SHORT_SHA=$(git rev-parse --short HEAD)" >> $GITHUB_ENV

      - name: Log in to Docker Hub
        uses: # TODO: Find the official docker login-action
        with:
          # !!! IMPORTANT !!! - do not paste the login and token here directly!
          # secrest should never be stored in repo code. Check the "Using secrets in GitHub Actions" docs.
          username: # TODO: Pass your secret here.
          password: # TODO: Pass your secret here.

      - name: Build Docker image (local)
        uses: # TODO: Find the official docker build-push-action
        with:
          context: .
          load: true # CRITICAL: This keeps the image in the runner so Trivy can scan it
          tags: # TODO: Format tag as -> ${{ secrets.DOCKERHUB_USERNAME }}/tsd-tutorial:yourname-${{ env.SHORT_SHA }}

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: # TODO: Pass the exact same tag you just generated above!
          format: 'table'
          exit-code: '1' # Fails the pipeline if vulnerabilities are found
          # TODO: Check Trivy docs to configure 'vuln-type' and set 'severity' to 'CRITICAL,HIGH'

      - name: Push Docker image
        uses: # TODO: Use the build-push-action again!
        with:
          context: .
          push: true # Now we actually push it!
          tags: # TODO: Pass the exact same tag again
```

---

## 📦 How to show your work?
- Merge your workflow into `main` (in your fork).
- Push a commit to `main`.
- In the **Actions** tab, show a successful run that:
  1) builds the image
  2) scans it
  3) pushes it to Docker Hub
- Verify on Docker Hub that your image tag exists and includes your name.
