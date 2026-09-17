# Pushing this to GitHub

## First time

```bash
cd linkedin-ai-job-search

git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPO.git
git branch -M main
git push -u origin main
```

If the remote already has a commit (e.g. GitHub created a README for you) and the
push is rejected, overwrite it — there is nothing there worth keeping:

```bash
git push -u origin main --force
```

If git asks for a password, use a **personal access token**, not your GitHub
password: github.com → Settings → Developer settings → Personal access tokens →
Tokens (classic) → Generate new token → tick `repo` → copy it and paste it as the
password.

## After any change

```bash
git add -A
git commit -m "Describe what changed"
git push
```

## Before you push, check

- [ ] `n8n/workflow.json` exported from n8n (see `n8n/README.md`)
- [ ] `site/index.html` has your own webhook URL in `ENDPOINT`
- [ ] No API keys, bot tokens or spreadsheet IDs anywhere in the files
