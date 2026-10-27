# uv workspace transitive deps — repro

Minimal reproduction for a problem in [renovatebot/renovate#47482](https://github.com/renovatebot/renovate/pull/47482), which surfaces `uv.lock` transitive dependencies for security updates.

## The problem

A uv workspace has one `uv.lock` at the root and a `pyproject.toml` per member. Renovate extracts each `pyproject.toml` as its own package file, and each one finds the same root lock file through `findLocalSiblingOrParent()`.

The PR reports every locked package which the *current* package file doesn't declare as a transitive dependency. A package declared by one member is therefore reported as transitive by every other member, and by the root.

This workspace has two members:

- `packages/a` declares `requests`, which pulls in `idna`
- `packages/b` declares `idna` directly
- the root declares nothing

`uv.lock` holds `idna` 3.6, which [GHSA-jjg7-2v4v-x38h](https://github.com/advisories/GHSA-jjg7-2v4v-x38h) affects. One advisory on one locked package produces three updates in one branch:

```
DEBUG: 3 flattened updates found: idna, idna, idna
       "branchName": "renovate/pypi-idna-vulnerability"   (x3)
```

and the branch regenerates the same lock file three times, with commands which disagree:

```
"command": "uv lock --upgrade-package idna==3.15"   # packages/a, as a transitive dep
"command": "uv lock --upgrade-package idna"         # packages/b, as a declared dep
"command": "uv lock --upgrade-package idna==3.15"   # root, as a transitive dep
```

An 8-package lock in a 3-member workspace is extracted as 13 transitive dependencies, so each one is also looked up once per member.

## Why the order matters

Here the last command is a pinned one, so the lock lands on the version Renovate chose:

```
$ uv lock --upgrade-package idna==3.15 && uv lock --upgrade-package idna==3.15
idna 3.15
```

If the member which *declares* the package sorts last, the unpinned command runs last instead, and the lock lands on the newest release rather than the chosen fix:

```
$ uv lock --upgrade-package idna==3.15 && uv lock --upgrade-package idna
idna 3.20
```

So whether `vulnerabilityFixStrategy` is honoured depends on package file iteration order.

## Reproduce

```bash
git clone https://github.com/khvn26/renovate-uv-workspace-transitive-repro
cd renovate-uv-workspace-transitive-repro
LOG_LEVEL=debug node /path/to/renovate/lib/renovate.ts --platform=local | grep 'flattened updates'
```

A full run against this repository produced [#1](https://github.com/khvn26/renovate-uv-workspace-transitive-repro/pull/1), which is correct in the end — `uv.lock` only, `idna` 3.6 to 3.15 — but took three `uv lock` runs to get there.

## A fix

`uv.lock` carries everything needed to tell declared from transitive across the whole workspace:

```toml
[manifest]
members = ["pkg-a", "pkg-b", "root"]

[[package]]
name = "pkg-a"
source = { virtual = "packages/a" }
dependencies = [
    { name = "requests" },
]
```

so transitive dependencies can be surfaced once, for the package file which owns the lock, with the union of all members' declared dependencies subtracted.
