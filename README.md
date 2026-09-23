# Reusable deploy workflows

The two workflows the Sundberg Solutions app repos call, and nothing else.

| | |
|---|---|
| [`deploy-static.yml`](.github/workflows/deploy-static.yml) | archetype A — build, three-pass `aws s3 sync`, CloudFront invalidation |
| [`deploy-ec2.yml`](.github/workflows/deploy-ec2.yml) | archetype B — SSH to the shared box, `docker compose up -d --build` |

## Why this repo is public, and separate

The documentation lives in **theted/infrastructure**, which is private and stays private: it holds
the AWS account number, every bucket and distribution id, the box's address and its port registry.

GitHub will not let a **public** repository call a reusable workflow that lives in a **private**
one — the call fails before any job starts, with `workflow was not found`. Every app repo in
`projects.json` is public, so a reusable workflow kept in the private repo could not be called by
any of them. Splitting the two files out here is what makes `uses:` work at all.

Nothing here is sensitive. A reusable workflow names the secrets it needs; it never contains them,
and the caller supplies them with `secrets: inherit` from its own repository settings.

## Calling them

```yaml
jobs:
  deploy:
    uses: theted/workflows/.github/workflows/deploy-static.yml@master
    with:
      build-command: npm run build
    secrets: inherit
```

Each file's header comment lists its inputs and the secrets it expects. Pin to `@master`: these
are read at the moment a run starts, so a fix here reaches every caller on their next deploy.

## Pinned actions

Every `uses:` inside these workflows names a commit sha, not a tag, with the release it belongs to
in a trailing comment. A moving tag is resolved fresh on each run, so a file that nobody has touched
can start behaving differently overnight — across all three sites at once, since they share this
repo. `.github/dependabot.yml` raises a weekly PR when a pinned action has a newer release, so the
pins move deliberately and land in one reviewable diff.

`uses: theted/workflows/...@master` in the *calling* repos is deliberately not pinned. That one is
ours, and a fix here should reach every caller on their next deploy.

## Changing one

Both workflows deploy live sites, and a mistake reaches every caller at once. Land the change,
then push a no-op commit to one calling repo — worldtemp is the usual canary — and watch that run
go green before assuming the rest will.
