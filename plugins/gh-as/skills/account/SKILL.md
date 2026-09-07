---
name: account
description: Run a gh command as the GitHub account a repository actually belongs to, on a machine where several accounts are logged in with gh. Use when gh reports "Could not resolve to a Repository", 404 or 403 for a repository that exists, when a push or PR must be made as a particular account, or before any gh command in a repository owned by someone other than the active account.
---

# Use the account the repository belongs to

`gh` keeps ONE active account per host in `~/.config/gh/hosts.yml`. When more
than one account is logged in, every command aimed at a repository the active
account cannot see fails as if the repository did not exist:

```
GraphQL: Could not resolve to a Repository with the name 'acme/service'. (repository)
```

Read that as an authentication mismatch, not a missing repository. The same
cause shows up as a bare 404 from `gh api` and as 403 on a push.

## 1. Resolve the command

- `gh-as` on `PATH` (`command -v gh-as`) — call it directly.
- Otherwise `gh extension list` showing `gh-as` — call it as `gh as`.
- Neither: tell the user to install it and stop —
  `gh extension install lambdalisue/gh-as`.

## 2. Rerun through it

```sh
gh-as gh pr list                          # account resolved from the origin remote
gh-as <account> -- gh pr create           # account named explicitly
gh-as --list                              # accounts logged in on the host
gh-as --print                             # the account that would be used
```

The wrapped command may be anything that reads `GH_TOKEN`, not only `gh`. The
chosen account is exported as `GH_AS_ACCOUNT`, and the token lives only in that
one process.

When resolution fails, `gh-as` says so rather than guessing. Either name the
account for that one command, or offer to record it for the repository:

```sh
git config --global gh-as.https://github.com/acme.account alice-work
```

## Never switch the active account

Do NOT run `gh auth switch` to get past one of these failures, and do not
suggest it. `hosts.yml` is global: the switch reaches every shell, editor and
agent on the machine, a concurrent command picks it up mid-flight, and a
command that dies before switching back leaves the user's machine pointing at
an account they did not choose. That is exactly the failure `gh-as` exists to
avoid.

`gh auth login`, `switch` and `logout` also refuse to run under an injected
token, so never wrap them — run them directly.

## Plain git needs no wrapper

git resolves credentials from its own URL-keyed `[credential]` sections and the
identity from `includeIf`, both per repository. Wrapping `git` in `gh-as`
changes nothing. When a git operation authenticates as the wrong user, fix the
git configuration instead of reaching for this skill.
