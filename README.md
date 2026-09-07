# gh-as

Run a command as one of the GitHub accounts `gh` is logged in with, without
switching the account the rest of the machine sees.

```console
$ cd ~/src/acme/service
$ gh pr list
GraphQL: Could not resolve to a Repository with the name 'acme/service'. (repository)

$ gh as -- gh pr list
gh-as: running as alice-work
Showing 3 of 3 open pull requests in acme/service
```

## The problem

`gh` keeps one active account per host in `~/.config/gh/hosts.yml`. With a
personal and a work account both logged in, every command aimed at a
repository the active account cannot see fails as if the repository did not
exist — the error names the repository, never the authentication.

The documented cure is `gh auth switch`, and it is a bad fit for the situation
that causes the error. `hosts.yml` is global: the switch reaches every shell,
editor, agent and background job on the machine. Two terminals working on
different accounts race over it, and a command that dies between the switch and
the switch back leaves the machine pointing somewhere its owner did not choose.

`gh-as` resolves the account, injects its token into the environment of one
command, and touches nothing else. Concurrent runs cannot interfere, and there
is no state to restore when a command fails.

## Install

As a `gh` extension:

```console
gh extension install lambdalisue/gh-as
```

Or drop the single script anywhere on `PATH`:

```console
curl -fsSLo ~/.local/bin/gh-as https://raw.githubusercontent.com/lambdalisue/gh-as/main/gh-as
chmod +x ~/.local/bin/gh-as
```

Both forms take the same arguments; the extension is spelled `gh as`, the
script `gh-as`.

## Usage

```console
gh-as <command> [args...]                # account resolved from the repository
gh-as <account> -- <command> [args...]   # account named explicitly
gh-as --print [<account>]                # print the account, run nothing
gh-as --list                             # accounts logged in on the host
```

The command is not limited to `gh` — anything that reads `GH_TOKEN` works:
`gh` extensions, a `curl` call against the API, your own release scripts.

```console
gh-as gh pr create
gh-as alice-work -- gh api user --jq .login
gh-as alice -- ./scripts/publish-release.ts
```

`GH_AS_ACCOUNT` carries the chosen account into the command, which makes it
easy to show in a prompt or assert in a script.

## Account resolution

Without an explicit account, the first of these that answers wins:

1. `git config --get-urlmatch gh-as.account <remote url>`
2. `git config --get-urlmatch credential.username <remote url>`
3. the repository owner, when it is itself a logged-in account
4. the only account logged in on the host

Steps 3 and 4 cover the common cases with no configuration at all: personal
repositories under your own name, and machines with a single login.

For anything else, declare the account per URL. git resolves the most specific
section, so an organization overrides the host-wide default:

```ini
[gh-as "https://github.com"]
  account = alice

[gh-as "https://github.com/acme"]
  account = alice-work
```

If you already pin credentials per URL, step 2 reads the account straight out
of that mapping and no `gh-as` section is needed.

## GitHub Enterprise

The host comes from the remote URL in resolution mode, from `--host` or
`GH_HOST` otherwise. Off `github.com` the token is passed as
`GH_ENTERPRISE_TOKEN` and `GH_HOST` is set for the command, matching how `gh`
itself reads enterprise credentials.

## Reference

| Option | |
| --- | --- |
| `-l`, `--list` | List the accounts logged in on the host |
| `-p`, `--print` | Print the account that would be used, then exit |
| `-q`, `--quiet` | Do not announce the account on stderr |
| `--host HOST` | Host to take the account from |
| `-h`, `--help` | Show help |

| Environment | |
| --- | --- |
| `GH_AS_REMOTE` | Remote to resolve the account from (default: `origin`) |
| `GH_AS_QUIET` | Non-empty implies `--quiet` |
| `GH_HOST` | Default host, as elsewhere in `gh` |

`gh auth login`, `switch` and `logout` refuse to run while a token is injected,
so `gh-as` rejects them with that explanation instead of letting `gh` fail
obscurely. Run those directly.

Plain `git` needs no wrapper: git already resolves credentials from its own
URL-keyed `[credential]` sections and the identity from `includeIf`. Reach for
`gh-as` when `gh` is involved.

## Development

```console
./test/test.sh        # runs against a stub gh; no account, no network
shellcheck gh-as test/test.sh
```

## License

MIT
