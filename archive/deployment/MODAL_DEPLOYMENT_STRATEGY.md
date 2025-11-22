# Modal Deployment Strategy

## Problem

Should we auto-deploy Modal in the pre-push git hook?

**TL;DR: No, use smart reminders instead (Option 1) or CI/CD (Option 2).**

---

## ❌ Why NOT Auto-Deploy in Pre-Push

1. **Slow** - Modal deployment takes 30-60 seconds, making every push painful
2. **Fragile** - Push fails if Modal is down or you're offline
3. **Team friction** - Requires all devs to have Modal authentication set up
4. **Unnecessary** - Most pushes don't change Modal code
5. **Blocking** - Can't quickly push a hotfix if Modal deploy fails

---

## ✅ Option 1: Smart Reminder (Recommended)

**What it does:**
- Detects if Modal code changed
- Shows a reminder to deploy
- Optionally prompts to deploy now
- **Non-blocking** - push continues even if you skip

**Setup:**

```bash
# 1. Configure git to use custom hooks directory
git config core.hooksPath .githooks

# 2. The hook is already created at .githooks/pre-push

# 3. Test it
git push
```

**What you'll see:**

If Modal code changed:
```
⚠️  ============================================
⚠️  Modal code has changed!
⚠️  ============================================

Changed files:
  - modal/services/vector_store.py
  - modal/modal_app.py

📦 Don't forget to deploy Modal after pushing:

   cd modal
   modal deploy modal_app.py

Deploy Modal now? (y/N):
```

If no Modal changes:
```
✅ [pre-push] No Modal changes detected
```

**Pros:**
- ✅ Fast (no deployment unless you want it)
- ✅ Helpful reminder when needed
- ✅ Optional deployment prompt
- ✅ Non-blocking

**Cons:**
- ⚠️ Requires discipline to actually deploy
- ⚠️ Easy to forget if you skip the prompt

---

## ✅ Option 2: GitHub Actions CI/CD (Best for Teams)

**What it does:**
- Automatically deploys Modal after successful push
- Only deploys if Modal code changed
- Everyone on team benefits, no local setup needed

**Setup:**

Create `.github/workflows/modal-deploy.yml`:

```yaml
name: Deploy Modal Functions

on:
  push:
    branches: [main, staging]
    paths:
      - 'modal/**'
      - '.github/workflows/modal-deploy.yml'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'

      - name: Install Modal
        run: pip install modal

      - name: Deploy Modal App
        env:
          MODAL_TOKEN_ID: ${{ secrets.MODAL_TOKEN_ID }}
          MODAL_TOKEN_SECRET: ${{ secrets.MODAL_TOKEN_SECRET }}
        run: |
          cd modal
          modal deploy modal_app.py

      - name: Comment on PR (if applicable)
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '✅ Modal functions deployed successfully!'
            })
```

**Add Modal secrets to GitHub:**

1. Get Modal token:
   ```bash
   modal token get
   ```

2. Add to GitHub:
   - Go to Settings → Secrets and variables → Actions
   - Add `MODAL_TOKEN_ID` and `MODAL_TOKEN_SECRET`

**Pros:**
- ✅ Fully automated
- ✅ No local setup needed (except for token)
- ✅ Consistent deployments
- ✅ Works for entire team
- ✅ Can run tests before deploying

**Cons:**
- ⚠️ Requires GitHub Actions setup
- ⚠️ Deployment happens after push (not before)
- ⚠️ Need to manage Modal token securely

---

## ✅ Option 3: Manual Deployment (Simplest)

**What it does:**
- Nothing automatic
- You deploy Modal manually when needed

**Setup:**
```bash
# Just remember to run after Modal code changes:
cd modal
modal deploy modal_app.py
```

**Pros:**
- ✅ Simplest approach
- ✅ Full control
- ✅ No infrastructure needed

**Cons:**
- ❌ Easy to forget
- ❌ Requires discipline
- ❌ Team members might forget

---

## ❌ Option 4: Force Deploy in Pre-Push (Not Recommended)

This is what you asked about - deploying Modal in pre-push hook automatically.

**Why it's bad:**

```bash
git push
# [pre-push] Deploying Modal...
# ⏰ Waiting 45 seconds...
# ❌ Modal deployment failed (offline)
# ❌ Push aborted!
#
# You: "I just wanted to push a README update! 😭"
```

**Only use this if:**
- You're a solo developer
- You always have internet
- Modal code changes very frequently
- You're okay with slow pushes

---

## Recommended Setup by Team Size

### Solo Developer
→ **Option 1** (Smart Reminder) or **Option 3** (Manual)

### Small Team (2-5 people)
→ **Option 2** (GitHub Actions)

### Larger Team (5+)
→ **Option 2** (GitHub Actions) + deployment tracking

---

## Implementation Guide

### For Option 1 (Smart Reminder):

```bash
# Enable the pre-push hook
git config core.hooksPath .githooks

# Test it
touch modal/test.py
git add modal/test.py
git commit -m "test: modal change detection"
git push

# Should show:
# ⚠️  Modal code has changed!
# Deploy Modal now? (y/N):
```

### For Option 2 (GitHub Actions):

```bash
# 1. Create workflow file
mkdir -p .github/workflows
cp MODAL_DEPLOYMENT_STRATEGY.md .github/workflows/modal-deploy.yml
# (edit the file to use the YAML above)

# 2. Get Modal token
modal token get

# 3. Add to GitHub Secrets
# Settings → Secrets → New repository secret
# Name: MODAL_TOKEN_ID
# Value: <from modal token get>
#
# Add another:
# Name: MODAL_TOKEN_SECRET
# Value: <from modal token get>

# 4. Push and test
git add .github/workflows/modal-deploy.yml
git commit -m "ci: add Modal auto-deployment"
git push

# GitHub Actions will deploy Modal automatically!
```

### For Option 3 (Manual):

```bash
# Just deploy when you change Modal code:
cd modal
modal deploy modal_app.py

# Consider creating an alias:
alias modal-deploy="cd modal && modal deploy modal_app.py && cd -"
```

---

## Decision Matrix

| Scenario | Recommended Option |
|----------|-------------------|
| Solo dev, changes Modal frequently | Option 1 (Reminder) |
| Solo dev, rarely changes Modal | Option 3 (Manual) |
| Team of 2-5 | Option 2 (GitHub Actions) |
| Team of 5+ | Option 2 (GitHub Actions) |
| Enterprise with strict controls | Option 2 + staging → prod workflow |
| Need offline development | Option 3 (Manual only when online) |

---

## Current Setup

I've created **Option 1** (Smart Reminder) for you at:
- `.githooks/pre-push` - Hook that checks for Modal changes

To enable it:
```bash
git config core.hooksPath .githooks
```

You can also add **Option 2** (GitHub Actions) on top for double coverage.

---

## Testing Your Setup

### Test the pre-push hook:

```bash
# Enable hooks
git config core.hooksPath .githooks

# Make a Modal change
echo "# test" >> modal/README.md
git add modal/README.md
git commit -m "test: modal hook"

# Push (should show reminder)
git push
```

### Test GitHub Actions:

```bash
# Push a Modal change to main
git push origin main

# Watch the action
gh workflow run modal-deploy.yml --ref main
gh run watch
```

---

## Summary

**My Recommendation: Use Option 1 (Smart Reminder)**

It's the sweet spot:
- Fast (non-blocking)
- Helpful (reminds when needed)
- Flexible (optional deploy prompt)
- Simple (no CI/CD setup)

Add **Option 2** (GitHub Actions) later if:
- Your team grows
- You want automation
- You have CI/CD already

**Don't** use auto-deploy in pre-push - it'll slow you down and break offline work.

---

## Files Created

- `.githooks/pre-push` - Smart reminder hook
- `MODAL_DEPLOYMENT_STRATEGY.md` - This document

**To enable:**
```bash
git config core.hooksPath .githooks
```

**To test:**
```bash
# Make a change to Modal code and push
echo "# test" >> modal/README.md
git add modal/README.md && git commit -m "test" && git push
```
