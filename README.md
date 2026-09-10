# ggshield GG-01 reproduction

This harmless repository demonstrates that ggshield 1.54.0 automatically loads
a tracked repository `.env` and passes its variables to ggshield and Git. The
`.env` selects a repository-controlled auth cache and a tracked executable
through `GIT_EXTERNAL_DIFF`. A normal pre-commit scan trusts the cache and then
executes that file while ggshield obtains the staged diff.

## Prerequisites

- Git
- An installed ggshield 1.54.0

No GitGuardian account or API credential, Docker, plugin, custom Git
configuration, custom server, or elevated privileges are required.

## Reproduce

Run these commands from outside any existing checkout:

```sh
git clone https://github.com/Jakick/ggshield-gg01-poc.git
cd ggshield-gg01-poc
printf '%s\n' 'changed by triager' >> application.txt
git add application.txt
ggshield secret scan pre-commit
cat .ggshield-gg01-executed
```

Expected marker:

```text
repository-controlled external diff executed
uid=1000(triager) gid=1000(triager) groups=1000(triager)
```

The exact user and group values vary. The second line is the output of the
repository-controlled `id` command, proving command execution with the
ggshield user's privileges. The marker is ignored by Git and can be removed
safely. The helper performs no network access and makes no change outside this
checkout.

## Why this is security-relevant

The `.env`, cache entry, and executable helper survive cloning as
repository-controlled content. ggshield loads every `.env` variable with
overriding semantics. This redirects its own authentication cache to the
repository and supplies the matching cache key. After trusting that cache,
ggshield invokes `git diff --staged`; Git interprets `GIT_EXTERNAL_DIFF` and
executes the helper with the developer's privileges.
