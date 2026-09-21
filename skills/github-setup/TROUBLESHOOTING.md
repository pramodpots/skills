# GitHub setup troubleshooting

- `gh: command not found` after brew install → open a new terminal, or `eval "$(/opt/homebrew/bin/brew shellenv)"`.
- `Permission denied (publickey)` → key not uploaded (`gh ssh-key list`) or not loaded (`ssh-add -l`; re-run `ssh-add --apple-use-keychain <key>`).
- Existing remotes still on HTTPS → `git remote set-url origin git@github.com:<user>/<repo>.git`.
- Wrong `gh` (name collision with another "gh" package) → `which gh` should be `/opt/homebrew/bin/gh`.
- Agents on CI or servers need a limited token → create a fine-grained token at github.com/settings/personal-access-tokens and export it as `GH_TOKEN`.
- `gh repo delete` fails on scope → `! gh auth refresh -h github.com -s delete_repo`, or delete the repo at `github.com/<username>/<repo>/settings`.
- `rm -rf` denied → hand the user `! rm -rf <path>`.
