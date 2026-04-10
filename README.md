# mirror

Experiment in mirroring ROOST repos to alternate git forges

## Configuring

This repo is powered by a simple [GitHub Actions workflow](.github/workflows/mirror.yml). It works by:

1. Creating a job for every repository included in the `repo` array
2. Checking out the repo and all of its history
3. Adding a remote pointing to the mirror, using a repository secret as the access token
4. Force-pushing all branches and tags to that mirror remote, explicitly pruning deleted branches
