# Git submodules primer

How this umbrella repository uses git submodules to track the Livepeer Modules, and how
to keep it updated as the network manages releases.

If you only read one thing: **this repo pins each module to a specific commit.** A
submodule is a pointer to one exact revision of another repo. Updating the suite means
deliberately moving those pointers and committing the move — never "whatever is on main
today."

## Mental model

- A submodule is a record in [`.gitmodules`](../../.gitmodules) (path + URL) plus a
  **gitlink**: a tree entry in this repo that stores the *commit SHA* of the module.
- Cloning this repo does **not** automatically fetch submodule contents — you ask for
  them explicitly (`--recursive` or `submodule update`).
- When you `cd modules/<name>` you are inside a normal, independent git repo checked
  out at the pinned SHA (usually in "detached HEAD" — that's expected).

Layout in this repo (per the suite convention): every module lives at
`modules/<name>/`.

## Cloning the suite (for a fresh checkout)

```bash
# Clone and fetch all submodules in one step
git clone --recurse-submodules <suite-repo-url>

# If you already cloned without submodules:
git submodule update --init --recursive
```

`--init` creates the local submodule from `.gitmodules`; `--recursive` handles
submodules-of-submodules (if any).

## Adding a module (onboarding a new repo)

When the user hands over a module repository:

```bash
# From the repo root. Pick the path that matches the module name.
git submodule add <module-repo-url> modules/<name>

# (Optional but recommended) pin to a released tag rather than the tip of main:
cd modules/<name>
git fetch --tags
git checkout <release-tag>        # e.g. v1.4.0
cd ../..

git add .gitmodules modules/<name>
git commit -m "Add <name> module submodule pinned to <release-tag>"
```

Then fill in `docs/product-specs/<name>.md` and update the index tables (see
[`../product-specs/index.md`](../product-specs/index.md)).

## Pinning to a network release

The whole point of the suite is reproducibility: the docs should describe a **known set
of module releases**. Prefer pinning each submodule to a **tag** that corresponds to a
network release, not to a moving branch.

```bash
cd modules/<name>
git fetch --tags origin
git checkout <release-tag>        # the released version you want to track
cd ../..
git add modules/<name>
git commit -m "Pin <name> to <release-tag>"
```

Record the revision you documented against in that module's spec.

## Updating modules as new releases ship

To bump one module to a newer release:

```bash
cd modules/<name>
git fetch --tags origin
git checkout <new-release-tag>
cd ../..
git add modules/<name>
git commit -m "Bump <name> to <new-release-tag>"
# then update docs/product-specs/<name>.md to reflect any changes
```

To pull the latest **on the tracked branch** for every submodule at once (use with
care — this moves pointers to branch tips, not tagged releases):

```bash
git submodule update --remote --merge
git add modules
git commit -m "Update submodules to latest tracked revisions"
```

You can set the branch a submodule tracks (used by `--remote`) in `.gitmodules`:

```bash
git config -f .gitmodules submodule.modules/<name>.branch <branch>
git add .gitmodules && git commit -m "Track <branch> for <name>"
```

## Syncing after someone else bumps a submodule

When you pull suite changes that moved submodule pointers, update your local checkouts:

```bash
git pull
git submodule update --init --recursive   # check out the SHAs the suite now points to
```

`git submodule update` moves your local submodule to the SHA recorded in the suite. It
does **not** advance to a module's latest commit — that's `--remote`.

## Inspecting state

```bash
git submodule status                  # show pinned SHA + checkout state per submodule
git submodule foreach 'git describe --tags || true'   # what release each is at
git diff --submodule                  # show submodule pointer moves in a diff
```

## Removing a module

```bash
git submodule deinit -f modules/<name>
git rm -f modules/<name>
rm -rf .git/modules/modules/<name>
git commit -m "Remove <name> submodule"
```

## Pitfalls

- **Forgetting to commit the pointer.** Checking out a new SHA inside a submodule does
  nothing for the suite until you `git add modules/<name>` and commit it.
- **Detached HEAD is normal.** Submodules check out a specific commit, not a branch.
  Only create/checkout a branch inside a submodule if you intend to develop there.
- **`update` vs `update --remote`.** `update` = "match the SHA the suite pins."
  `--remote` = "advance to the tracked branch's latest." Don't confuse them.
- **Stale clones.** After pulling, always run `git submodule update --init --recursive`
  or your `modules/` directories will lag the pointers.
- **Auth/URLs.** Use URLs every contributor can fetch. Switching SSH⇄HTTPS later means
  editing `.gitmodules` and running `git submodule sync`.

## Recommended workflow for this suite

1. Track each module by **tagged release** matching a network release.
2. Bump one module at a time; in the same commit, update its spec in
   `docs/product-specs/`.
3. Note the pinned revision in the module spec and the
   [product-specs index](../product-specs/index.md).
4. Treat a suite commit as a snapshot: "these exact module releases, documented this
   way." That's the artifact this repo exists to produce.
