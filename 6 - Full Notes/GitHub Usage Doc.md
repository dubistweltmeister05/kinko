[[RhyGen]]

# GitHub Development Workflow Guide
## Purpose
All development must utilize GitHub to its fullest potential to ensure code quality, accountability, traceability, and a clean, linear commit history.
This guide establishes:
- Authentication and security requirements
- Standard Git configuration
- Branching and pull request workflows
- Release management procedures using Git tags

---

# 1. Authentication & Security Setup
To interact with GitHub repositories from your local machine, configure either:
- **SSH Keys (Recommended)** – Secure and passwordless terminal operations.
- **Personal Access Tokens (PATs)** – Required when using HTTPS repository URLs.

---

## A. SSH Key Configuration (Recommended)
SSH keys provide secure authentication without requiring a username and password for every Git operation.

### Step 1: Generate a New SSH Key

Replace the email address below with your GitHub account email.
```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

When prompted:
- Press **Enter** to save the key to the default location:  
    `~/.ssh/id_ed25519`
    
- Optionally provide a secure passphrase.
---

### Step 2: Start the SSH Agent
```bash
eval "$(ssh-agent -s)"
```

---

### Step 3: Add Your SSH Key to the Agent
```bash
ssh-add ~/.ssh/id_ed25519
```

---

### Step 4: Add the Public Key to GitHub
Copy your public key:
```bash
cat ~/.ssh/id_ed25519.pub
```
Then:
1. Open GitHub.
2. Navigate to:  
    **Settings → SSH and GPG Keys → New SSH Key**
3. Paste the public key.
4. Save the key.

---

### Step 5: Verify the Connection
```bash
ssh -T git@github.com
```

A successful authentication message confirms setup is complete.

---

## B. Personal Access Tokens (PAT)
If you clone repositories using HTTPS URLs, GitHub requires a Personal Access Token instead of an account password.
### Step 1: Generate a Token
Navigate to:
**GitHub → Settings → Developer Settings → Personal Access Tokens → Tokens (Classic)**

1. Select **Generate New Token (Classic)**.
2. Add a descriptive name.
3. Set an expiration date.
4. Select the **repo** scope (minimum required).
5. Generate the token.
6. Copy and securely store the token.

> Important: GitHub only displays the token once.

---

### Step 2: Configure Credential Storage
Store credentials locally to avoid repeated authentication prompts.
```bash
git config --global credential.helper store
```

The next time Git prompts for credentials:
- Username = Your GitHub username
- Password = Your Personal Access Token
---

# 2. Global Git Configuration
To maintain a clean and linear commit history, all developers must use a **rebase pull strategy**.
Configure Git globally:
```bash
git config --global pull.rebase true
```
### Why Use Rebase?
When local commits exist that have not yet been pushed:
- `git pull` fetches the latest remote changes.
- Local commits are automatically replayed on top of those changes.
- The commit history remains linear.
- Unnecessary merge commits are avoided.

Example of commits we want to avoid:
```text
Merge branch 'main' of github.com/company/project
```

A linear history is easier to review, debug, and maintain.

---

# 3. Feature Development Lifecycle
Every feature, bug fix, enhancement, or maintenance task must follow the workflow below.

---

## Step 1: Synchronize Your Local Repository
Ensure your local `main` branch matches the remote repository.
```bash
git checkout main
git pull
```

---

## Step 2: Create a Dedicated Branch
Never develop directly on the `main` branch.
Use a descriptive naming convention:
- `feature/<feature-name>`
- `bugfix/<bug-name>`
- `chore/<task-name>`

Example:
```bash
git checkout -b feature/user-authentication
```

---

## Step 3: Develop and Commit Changes
After implementing and testing changes locally:
```bash
git add .
git commit -m "feat: added user authentication endpoints"
```

### Commit Message Guidelines
Use descriptive commit messages:
```text
feat: add authentication endpoints
fix: resolve login validation bug
chore: update dependencies
docs: update API documentation
refactor: simplify token handling logic
```

---

## Step 4: Push the Branch and Open a Pull Request

Push the feature branch to GitHub:

```bash
git push -u origin feature/user-authentication
```

Then:
1. Open the GitHub repository.
2. Create a Pull Request (PR).
3. Target the `main` branch.
4. Provide a clear description of the changes.

---

## Step 5: Peer Review and Iteration
### Review Requirements
- Every Pull Request must be reviewed.
- At least one team member must approve the PR before merging.
### Addressing Review Feedback
Make requested changes locally and push them to the same branch.

```bash
git add .
git commit -m "fix: update validation logic per PR review"
git push
```

The Pull Request updates automatically.

---

## Step 6: Merge to Main
After all approvals are received and checks pass:

1. Merge the Pull Request into `main`.
2. Delete the feature branch if no longer needed.
3. Pull the latest changes locally.

```bash
git checkout main
git pull
```

---

# 4. Release Management Using Git Tags

Git tags identify important milestones and release versions.

Examples:

- `v1.0.0`
- `v1.1.0`
- `v2.0.0`

Unlike branches, tags are static references and do not move as development continues.

---

## Benefits of Tags

### Traceability
Quickly locate the exact code used for a release.
### Deployment Consistency
Deploy the same version repeatedly without ambiguity.
### Historical Auditing
Review source code exactly as it existed at a specific point in time.

---

## Creating Annotated Tags

Always use annotated tags (`-a`) because they record:
- Creator    
- Timestamp
- Message
- Version information

### Tag the Current Commit
```bash
git tag -a v1.0.0 -m "Release version 1.0.0 production ready"
```

---

### Tag a Historical Commit

Locate the commit hash using:
```bash
git log
```

Then create the tag:
```bash
git tag -a v1.0.0 <commit_hash> -m "Tagged the historical first release version 1.0.0"
```

---

## Pushing Tags to GitHub
Tags are not pushed automatically.
### Push a Single Tag
```bash
git push origin v1.0.0
```
### Push All Tags
```bash
git push origin --tags
```

---

# Quick Reference Workflow
```text
1. Checkout main
2. Pull latest changes
3. Create feature branch
4. Develop and test
5. Commit changes
6. Push branch
7. Open Pull Request
8. Address review feedback
9. Obtain approvals
10. Merge to main
11. Create release tag (if applicable)
12. Push tag to GitHub
```

---

# Team Standards Summary

✓ Never commit directly to `main`
✓ Always create a feature, bugfix, or chore branch
✓ Use descriptive commit messages
✓ Keep Pull Requests focused and reviewable
✓ Require at least one approval before merging
✓ Use `git pull` with rebase enabled
✓ Create annotated tags for releases
✓ Push release tags to GitHub
✓ Maintain a clean, linear commit history