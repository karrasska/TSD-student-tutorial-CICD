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

## 💡 Hints (Step-by-Step Guide)

To build your workflow, you need to combine a few standard GitHub Actions. Think of them as Lego blocks.

**Step 1: The Skeleton**
Start your file by defining the name, the trigger, and the job:
```yaml
name: CI Pipeline
on: [pull_request]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      # Your steps will go here
```

**Step 2: Fetch Code and Setup Python**
Under `steps:`, you always need to check out your code first, then install the language environment. Use these exact actions:
```yaml
      - name: Check out code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'
          cache: 'pip' # This speeds up your runs!
```

**Step 3: Install and Run**
Now add standard run commands just like you would in your terminal:
```yaml
      - name: Install dependencies
        run: pip install -r requirements-dev.txt

      - name: Run Formatting Check
        run: black --check .
        
      # Now add your own steps for 'ruff check .' and 'pytest'!
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
*(Note: CodeQL requires `security-events: write` permissions, which the generated file handles automatically).*

---

### (B) Dependabot (automatic dependency update PRs)

**💡 How to do it:** Dependabot doesn't use standard workflows. It uses a specific config file.
1. Create `.github/dependabot.yml`.
2. Use this template and fill in the missing `directory` and `schedule` details based on the official docs:
```yaml
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/" # Where are your requirements files?
    schedule:
      interval: "weekly" 
```

---

### (C) Automatic PR comment when CI succeeds

**💡 How to do it:** 1. Open your `ci.yml` from Task 1.
2. Add a **second job** right below your first one. 
3. Use the `needs` keyword so it only runs if the tests pass, and use `actions/github-script` to interact with the PR:

```yaml
  pr-comment:
    needs: build-and-test # Must match the name of your first job!
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write # Required to post a comment
    steps:
      - name: Comment on PR
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '✅ CI passed: tests + lint + formatting are green!'
            })
```

---

## 📦 How to show your work?
- Open a PR with the new files:
  - `.github/workflows/codeql.yml`
  - `.github/dependabot.yml`
  - updated `.github/workflows/ci.yml` (adds PR comment job)
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
2. Add `DOCKERHUB_USERNAME`.
3. Add `DOCKERHUB_TOKEN` (provided during class).

---

### (B) Setup the Workflow Skeleton
1. Create `.github/workflows/docker.yml`.
2. Trigger it **only** on `push` to `main` (never `pull_request` for deployments!).

### (C & D) Generate Tags, Build, and Authenticate

**💡 Step-by-Step Guide for the Job:**

**1. Create a variable for your commit SHA** so you can tag your image uniquely:
```yaml
      - name: Get Short SHA
        run: echo "SHORT_SHA=$(git rev-parse --short HEAD)" >> $GITHUB_ENV
```

**2. Log in to DockerHub** using the secrets you created:
```yaml
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
```

**3. Build the Image (but don't push yet!)** We need the image built locally so Trivy can scan it. Notice `load: true`:
```yaml
      - name: Build Docker image (local)
        uses: docker/build-push-action@v5
        with:
          context: .
          load: true # Keeps image in the runner for scanning
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/shared-repo:yourname-${{ env.SHORT_SHA }}
```

### (E) Container scanning & Final Push

**4. Scan the image with Trivy.** Use the official Aqua Security action. Tell it to look at the exact tag you just built:
```yaml
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: '${{ secrets.DOCKERHUB_USERNAME }}/shared-repo:yourname-${{ env.SHORT_SHA }}'
          format: 'table'
          exit-code: '1' # Fails the pipeline if vulnerabilities are found!
          ignore-unfixed: true
          vuln-type: 'os,library'
          severity: 'CRITICAL,HIGH'
```

**5. Push the Image.**
If Trivy passes, the pipeline will continue. Now you can safely push!
```yaml
      - name: Push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true # Now we actually push it!
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/shared-repo:yourname-${{ env.SHORT_SHA }}
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
