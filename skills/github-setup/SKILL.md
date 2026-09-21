---
name: github-setup
description: Set up GitHub on a Mac from scratch, including creating a GitHub account if needed, so AI agents can create repos, push branches, open and review PRs with no password prompts. Installs git and the gh CLI, configures identity, logs in, sets up an SSH key, then verifies everything with a throwaway repo and PR. Use when the user asks to set up GitHub, install gh, connect GitHub for agents, or fix GitHub auth/push problems.
---

# GitHub setup for AI agents (macOS)

Goal: the user ends with `git` + `gh` installed, logged in, pushing over **SSH** (no repeated password logins), verified by a real end-to-end test.

## Ground rules

- **Check before installing.** Every step starts with a check; skip it if already done. Tell the user what was already set up.
- **You can't do interactive/browser/password steps.** Give the user the exact command prefixed with `!` (so it runs in this session), say what they will see, and wait for them to say "done". Never ask for or handle passwords or tokens in chat.
- **Outward-facing or destructive actions need confirmation**: creating the test repo, deleting it, deleting local files. Ask first, one action per command, no long `&&` chains that mix create and delete.
- Use plain single-purpose commands. If a command is denied, don't retry verbatim; explain and offer the user a `!` command instead.

## Step 0: GitHub account (beginners start here)

Ask: "Do you already have a GitHub account?" If yes, skip to Step 1. If not or unsure, walk them through it one step at a time, waiting for "done" between steps:

> 1. Open https://github.com/signup in your browser.
> 2. Enter your email, create a password, and pick a **username**. It becomes part of your public URLs (github.com/username), so choose something you're happy to share.
> 3. Solve the puzzle and enter the verification code GitHub emails you.
> 4. Skip any optional questions and choose the **Free** plan.
> 5. **Turn on two-factor authentication** (avatar > Settings > Password and authentication). GitHub requires it, and an authenticator app is the easiest way.
> 6. Tell me your username and the email you signed up with.

Use that same email in Step 3 (or their noreply address). Never ask for their password.

## Step 1: Inspect current state

```bash
which brew git gh; git --version; gh --version
gh auth status 2>&1
git config --global user.name; git config --global user.email
ls ~/.ssh
ssh -T -o StrictHostKeyChecking=accept-new git@github.com 2>&1 | head -3
```

Summarize what's present and what's missing, then continue with only the missing steps.

## Step 2: Install tools

**Homebrew missing** — needs the user's Mac password, so the user runs it:

> Paste this in the prompt (with the `!`), enter your Mac password when asked, and follow any "Next steps" it prints (usually two `eval` lines to add brew to your PATH):
> `! /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`

**git missing** — `brew install git` (or `xcode-select --install`, which opens a GUI installer the user must click through).

**gh missing** — `brew install gh`

Re-run `git --version` and `gh --version` to confirm.

## Step 3: Git identity

If `user.name` / `user.email` are empty, ask the user for their name and their GitHub email (or the `ID+username@users.noreply.github.com` address from github.com/settings/emails if they want privacy), then:

```bash
git config --global user.name "Their Name"
git config --global user.email "their@email"
git config --global init.defaultBranch main
```

## Step 4: Log in to GitHub (browser, user does this)

Give the user this exact step:

> 1. Paste: `! gh auth login --web --git-protocol ssh --scopes repo,workflow,read:org,gist,admin:public_key`
> 2. If asked, choose **GitHub.com**. If it asks to generate/upload an SSH key, say **yes** and accept the defaults (an empty passphrase is fine, or use one and let the macOS Keychain remember it).
> 3. It prints a one-time code like `ABCD-1234`. Press Enter, your browser opens, paste the code, sign in, click **Authorize**.
> 4. Tell me "done".

Scopes: `repo` (repos/PRs), `workflow` (Actions files), `read:org`, `gist`, `admin:public_key` (upload the SSH key). Add `delete_repo` only if the user wants agents to be able to delete repos; it lets the test clean up after itself, so recommend it for the test and offer to revoke later with `gh auth refresh` (without the scope).

Verify: `gh auth status`

## Step 5: SSH key (no password prompts)

Check whether GitHub already accepts the machine's key:

```bash
ssh -T -o StrictHostKeyChecking=accept-new git@github.com 2>&1 | head -3
```

- `Hi <user>! You've successfully authenticated` → done, skip to Step 6.
- `Permission denied (publickey)` → a key is missing or not uploaded.

If no key exists (`ls ~/.ssh/id_ed25519.pub ~/.ssh/id_rsa.pub`), generate one. The user should pick a passphrase-free key only if they are comfortable with that; otherwise use the Keychain:

```bash
ssh-keygen -t ed25519 -C "their@email" -f ~/.ssh/id_ed25519 -N ""
```

Make macOS load the key automatically:

```bash
cat >> ~/.ssh/config <<'EOF'
Host github.com
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_ed25519
EOF
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```

(Skip the config append if `~/.ssh/config` already has a `Host github.com` block.) Upload it:

```bash
gh ssh-key add ~/.ssh/id_ed25519.pub --title "$(scutil --get ComputerName)"
```

If this fails on scope: ask the user to run `! gh auth refresh -h github.com -s admin:public_key` (browser flow like Step 4).

Then make `gh` and git use SSH and re-test:

```bash
gh config set git_protocol ssh -h github.com
ssh -T git@github.com 2>&1 | head -3
```

## Step 6: End-to-end test (confirm with the user first)

Tell the user: "I'll create a private throwaway repo, push a branch, open a PR, review it, then delete the repo. OK?" Wait for yes. Use a temp dir (not the user's projects). Run these as **separate commands**:

1. Create + push:
   ```bash
   mkdir -p ~/gh-setup-test && cd ~/gh-setup-test && git init -b main && echo hi > README.md && git add . && git commit -m init && gh repo create gh-agent-test --private --source=. --push
   ```
2. Branch + PR:
   ```bash
   git checkout -b feature && echo more >> README.md && git commit -am "feature change" && git push -u origin feature && gh pr create --title "Test PR" --body "test" --base main --head feature
   ```
3. Review (GitHub doesn't allow approving your own PR, so use a comment review):
   ```bash
   gh pr review 1 --comment --body "LGTM test review" && gh pr diff 1 && gh pr view 1 --json state,reviews --jq '{state,reviews:(.reviews|length)}'
   ```
   Expect `state: OPEN`, `reviews: 1`.

## Step 7: Clean up (confirm first)

Ask: "Test passed. Delete the throwaway repo and the local folder?" Then, separately:

```bash
gh repo delete <username>/gh-agent-test --yes     # needs delete_repo scope
rm -rf ~/gh-setup-test
```

If `gh repo delete` fails on scope: `! gh auth refresh -h github.com -s delete_repo`, or have the user delete it at `github.com/<username>/gh-agent-test/settings`. If `rm -rf` is denied, give the user `! rm -rf ~/gh-setup-test`.

## Step 8: Report

Give a short summary: what was already there, what you installed, auth method (SSH), scopes granted, test result, cleanup status. List what agents can now do: `gh repo create`, `gh pr create/review/diff/merge`, `gh issue`, `gh run`, plain `git push`.

## Troubleshooting

- `gh: command not found` after brew install → open a new terminal, or `eval "$(/opt/homebrew/bin/brew shellenv)"`.
- `Permission denied (publickey)` → key not uploaded (`gh ssh-key list`) or not loaded (`ssh-add -l`; re-run `ssh-add --apple-use-keychain <key>`).
- Existing remotes still on HTTPS → `git remote set-url origin git@github.com:<user>/<repo>.git`.
- Wrong `gh` (name collision with another "gh" package) → `which gh` should be `/opt/homebrew/bin/gh`.
- Need agents to use a limited token instead (CI/servers): create a fine-grained token at github.com/settings/personal-access-tokens and export it as `GH_TOKEN`.
