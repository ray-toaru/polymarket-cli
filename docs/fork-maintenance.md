# Fork maintenance workflow

This fork keeps upstream sync separate from custom changes.

## Branch roles

- `main`: mirror of `Polymarket/polymarket-cli:main` as closely as possible.
- `custom/main`: long-lived branch for this fork's custom behavior.
- `feat/*`: feature branches based on `custom/main`.
- `upstream/*`: clean contribution branches intended for upstream PRs.

## Sync upstream into the fork

```bash
git fetch upstream

git checkout main
git merge --ff-only upstream/main
git push origin main
```

Then replay custom changes onto the updated upstream baseline:

```bash
git checkout custom/main
git rebase main
git push --force-with-lease origin custom/main
```

Enable rerere locally to reduce repeated conflict work:

```bash
git config --global rerere.enabled true
```

## Develop custom changes

```bash
git checkout custom/main
git pull --rebase origin custom/main
git checkout -b feat/my-change
```

Open PRs into `custom/main`, not into `main`.

## Contribute a clean patch upstream

For upstreamable changes, create a dedicated branch from the clean `main` baseline:

```bash
git checkout main
git pull --ff-only origin main
git checkout -b upstream/my-change
```

Keep this branch free from fork-only behavior.
