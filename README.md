# mirror

A GitHub Action for mirroring a git repository to any git forge, plus a workflow that uses it to keep ROOST's repos in sync.

## Usage

After checking out your repo with `fetch-depth: 0`, call this action once per destination forge:

```yaml
- uses: actions/checkout@v6
  with:
    repository: your-org/your-repo
    fetch-depth: 0

- name: Mirror to GitLab
  uses: roostorg/mirror@main
  with:
    destination: https://gitlab.com/your-org/your-repo.git
    token: ${{ secrets.GITLAB_TOKEN }}

- name: Mirror to Tangled
  uses: roostorg/mirror@main
  with:
    destination: git@tangled.org:your-handle/your-repo
    ssh_private_key: ${{ secrets.TANGLED_SSH_PRIVATE_KEY }}
    ssh_known_hosts: ${{ secrets.TANGLED_KNOWN_HOSTS }}
```

The action mirrors all branches and tags, pruning any that no longer exist in the source. Mirrors should be treated as read-only; there is no two-way sync.

### Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `destination` | yes | — | Remote URL to push to (HTTPS or SSH). Do not embed credentials here. |
| `token` | for HTTPS | — | OAuth2 or personal access token. Injected into the URL; never echoed. |
| `token_username` | no | `oauth2` | Username in the HTTPS URL. `oauth2` works for GitLab; use the account username for Gitea, Forgejo, and Codeberg. |
| `ssh_private_key` | for SSH | — | PEM private key (ED25519 or RSA). Written to a temp file with mode 0600 and deleted after the push. |
| `ssh_known_hosts` | recommended | — | Known-hosts entries for the destination host (see below). If omitted, falls back to `ssh-keyscan` with a TOFU warning. |

### Forge examples

**GitLab** — HTTPS with OAuth2 token (default `token_username` of `oauth2` is correct):
```yaml
- uses: roostorg/mirror@main
  with:
    destination: https://gitlab.com/your-org/your-repo.git
    token: ${{ secrets.GITLAB_TOKEN }}
```

**Tangled** — SSH with ED25519 key:
```yaml
- uses: roostorg/mirror@main
  with:
    destination: git@tangled.org:your-handle/your-repo
    ssh_private_key: ${{ secrets.TANGLED_SSH_PRIVATE_KEY }}
    ssh_known_hosts: ${{ secrets.TANGLED_KNOWN_HOSTS }}
```

**Codeberg** — HTTPS with a Codeberg username and token:
```yaml
- uses: roostorg/mirror@main
  with:
    destination: https://codeberg.org/your-username/your-repo.git
    token: ${{ secrets.CODEBERG_TOKEN }}
    token_username: your-username
```

**Gitea / Forgejo** — HTTPS with account username and token:
```yaml
- uses: roostorg/mirror@main
  with:
    destination: https://your-forgejo-instance.example.com/your-username/your-repo.git
    token: ${{ secrets.FORGEJO_TOKEN }}
    token_username: your-username
```

### Getting `ssh_known_hosts`

Run this once and store the output as a repository secret:

```sh
ssh-keyscan -t ed25519,rsa,ecdsa tangled.org
```

Providing this value means the action verifies the host's identity at push time. Without it, the action falls back to `ssh-keyscan` on each run, which does not verify host identity (trust-on-first-use).

### Security notes

- Tokens are injected into a shell variable and never echoed; GitHub Actions also masks the raw secret value in all log output.
- SSH private keys are written to `$RUNNER_TEMP` (not `~/.ssh`) with `chmod 600` and deleted immediately after the push.
- `IdentitiesOnly=yes` prevents the SSH agent from offering other keys during the push.
- `StrictHostKeyChecking=yes` ensures the action hard-fails if the host key doesn't match, rather than silently connecting.
- Only the git repository (branches, tags, and commits) is mirrored — pull requests and issues are not.
- **Do not mirror private repositories.** The action does not check repo visibility.

## How ROOST uses this

ROOST maintains a [mirror workflow](.github/workflows/mirror.yml) in this repo that runs hourly and on push to `main`. It checks out each repo in a matrix, then calls this action twice — once for GitLab, once for Tangled.

To add a ROOST repo to the mirror:

1. Create an empty repo of the same name on each forge (e.g. [gitlab.com/roostorg](https://gitlab.com/roostorg/))
2. Add the repo name to the `repo` matrix in [.github/workflows/mirror.yml](.github/workflows/mirror.yml)
3. Wait up to an hour, or trigger the workflow manually from the Actions tab
