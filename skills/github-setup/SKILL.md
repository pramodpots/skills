---
name: github-setup
description: GitHub setup on a Mac, from zero (account, git, gh CLI, SSH) to a verified agent-ready workflow, and repair of broken GitHub auth or push failures. Triggers - set up GitHub, install gh, connect GitHub for agents, permission denied on push.
---

# GitHub setup for AI agents (macOS)

Done when `git` and `gh` are installed, logged in, pushing over **SSH** with no password prompts, and a real end-to-end test has passed.

## Ground rules

- **Check first.** Each step opens with a check; skip the step when it already passes, and tell the user what was already in place.
- **Hand off browser, password and sudo steps.** Give the user the exact command prefixed with `!` (it runs in this session), say what they will see, and wait for "done". Passwords and tokens are typed by the user into the browser or terminal prompt, never into chat.
- **Gate every outward-facing or destructive action.** Creating the test repo, deleting it, and deleting local files each need the user's yes first.
- **One action per command.** Create and delete never share a command chain. When a command is denied, explain why and offer the user a `!` version.
- Failures: see [TROUBLESHOOTING.md](TROUBLESHOOTING.md) for command-not-found, publickey errors, HTTPS remotes, scope errors and denied commands.

## Step 0: GitHub account

Ask: "Do you already have a GitHub account?" Yes → Step 1. No or unsure → walk through these one at a time, waiting for "done" between them:

> 1. Open https://github.com/signup.
> 2. Enter your email, create a password, and pick a **username** (it appears in your public URLs, so choose one you'll share).
> 3. Solve the puzzle and enter the code GitHub emails you.
> 4. Skip the optional questions and choose the **Free** plan.
> 5. Turn on two-factor authentication (avatar > Settings > Password and authentication); an authenticator app is easiest.
> 6. Tell me your username and signup email.

Done when the user has given a username and email.

## Step 1: Inspect

```bash
which brew git gh; git --version; gh --version
gh auth status 2>&1
git config --global user.name; git config --global user.email
ls ~/.ssh
ssh -T -o StrictHostKeyChecking=accept-new git@github.com 2>&1 | head -3
```

Done when you have told the user which of these are present and which are missing. Every later step covers only what is missing.

## Step 2: Install tools

**Homebrew** (needs the Mac password, so the user runs it):

> Paste this with the `!`, enter your Mac password when asked, then run the "Next steps" lines it prints (usually two `eval` lines that put brew on your PATH):
> `! /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`

**git**: `brew install git` (alternative: `xcode-select --install`, a GUI installer the user clicks through).
**gh**: `brew install gh`

Done when `git --version` and `gh --version` both print versions.

## Step 3: Git identity

Ask for the user's name and GitHub email (or the `ID+username@users.noreply.github.com` address from github.com/settings/emails for privacy). Then:

```bash
git config --global user.name "Their Name"
git config --global user.email "their@email"
git config --global init.defaultBranch main
```

Done when both `git config --global user.name` and `user.email` print values.

## Step 4: Log in (browser, user runs it)

> 1. Paste: `! gh auth login --web --git-protocol ssh --scopes repo,workflow,read:org,gist,admin:public_key,delete_repo`
> 2. Choose **GitHub.com**. If it offers to generate and upload an SSH key, say **yes** and accept the defaults.
> 3. It prints a one-time code like `ABCD-1234`. Press Enter, your browser opens, paste the code, sign in, click **Authorize**.
> 4. Tell me "done".

Scopes: `repo` (repos and PRs), `workflow` (Actions files), `read:org`, `gist`, `admin:public_key` (upload the SSH key), `delete_repo` (test cleanup; the broadest of the set, so the user may drop it and delete the test repo in the browser instead).

Done when `gh auth status` shows the user logged in.

## Step 5: SSH key

Runs only when Step 1's `ssh -T` did not answer `Hi <user>! You've successfully authenticated`; re-run it first, since Step 4 may already have uploaded a key.

If `ls ~/.ssh/id_ed25519.pub ~/.ssh/id_rsa.pub` finds no key, ask: "Passphrase on the key, or none?"

- **None**: `ssh-keygen -t ed25519 -C "their@email" -f ~/.ssh/id_ed25519 -N ""`
- **Passphrase**: the user runs `! ssh-keygen -t ed25519 -C "their@email" -f ~/.ssh/id_ed25519` and types the passphrase at the prompt.

Load it through the macOS Keychain (skip the `cat` when `~/.ssh/config` already has a `Host github.com` block):

```bash
cat >> ~/.ssh/config <<'EOF'
Host github.com
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_ed25519
EOF
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```

Upload and switch `gh` to SSH:

```bash
gh ssh-key add ~/.ssh/id_ed25519.pub --title "$(scutil --get ComputerName)"
gh config set git_protocol ssh -h github.com
```

Scope error on upload → the user runs `! gh auth refresh -h github.com -s admin:public_key`.

Done when `ssh -T git@github.com` answers `Hi <user>! You've successfully authenticated`.

## Step 6: End-to-end test

Gate: "I'll create a private throwaway repo, push a branch, open a PR, and review it. OK?" Then run these as **separate commands** in `~/gh-setup-test`, not in the user's projects:

1. Create and push:
   ```bash
   mkdir -p ~/gh-setup-test && cd ~/gh-setup-test && git init -b main && echo hi > README.md && git add . && git commit -m init && gh repo create gh-agent-test --private --source=. --push
   ```
2. Branch and PR:
   ```bash
   git checkout -b feature && echo more >> README.md && git commit -am "feature change" && git push -u origin feature && gh pr create --title "Test PR" --body "test" --base main --head feature
   ```
3. Review (GitHub blocks approving your own PR, so use a comment review):
   ```bash
   gh pr review 1 --comment --body "LGTM test review" && gh pr diff 1 && gh pr view 1 --json state,reviews --jq '{state,reviews:(.reviews|length)}'
   ```

Done when step 3 prints `state: OPEN` and `reviews: 1`.

## Step 7: Clean up

Gate: "Test passed. Delete the throwaway repo and the local folder?" Then, as two separate commands:

```bash
gh repo delete <username>/gh-agent-test --yes
rm -rf ~/gh-setup-test
```

Done when `gh repo view <username>/gh-agent-test` reports not found and `~/gh-setup-test` is gone.

## Step 8: Report

Summarize in a few lines: what was already in place, what you installed, auth method (SSH), scopes granted, test result, cleanup status. Close with what agents can now do: create repos, push, open, review and merge PRs, manage issues and Actions runs.
