# Act 4: Squash-first, stale-second PR — loss reproduction

Reproduces the scenario from your notes:

```text
Main:     A --- B --- C
               \
Your PR:        D --- E --- F   (stale after someone squash-merges S onto main)
```

Repo: [Disturbing/merge-test](https://github.com/Disturbing/merge-test)

## Scenario A — parallel stale PR (`loss-demo.txt`)

| Step | PR | Branch | What it does |
|------|-----|--------|----------------|
| 1 | [#7](https://github.com/Disturbing/merge-test/pull/7) | `act4-squasher` | Squash merge **first** → adds `squasher=PRESENT`, `line5=FROM_SQUASHER` |
| 2 | [#8](https://github.com/Disturbing/merge-test/pull/8) | `act4-yours-stale` | Only `line1=YOUR_PR`; branch still has `squasher=absent` |

**After #7 (main):**

```text
line1=hello
line5=FROM_SQUASHER
squasher=PRESENT
```

**#8 without updating:** GitHub → **`mergeable: CONFLICTING`** (merge and squash both blocked).

Git does **not** silently drop the squasher’s work. It stops and forces an explicit resolution.

### How the squasher *would* lose (manual wrong fix)

Local demo branch `act4-demo-wrong-merge` (commit `eb58fa1`):

```bash
git merge origin/act4-yours-stale   # conflict
git checkout --theirs loss-demo.txt # take stale snapshot
git commit
```

**Result — squasher’s work gone:**

```text
line1=YOUR_PR
line5=neutral
squasher=absent
```

So **yes**: the squasher loses their features if *your* stale PR is merged and someone resolves the conflict by taking **your** outdated file. That is a **human/process** failure, not a silent Git overwrite.

## Scenario B — dependent branch (`loss-demo.txt`)

Branch `act4-yours-dependent` = squasher commit **D** + your commit **F** (same edits as squashed **S**, plus `line1=YOUR_DEPENDENT`).

| PR | Result |
|----|--------|
| [#9](https://github.com/Disturbing/merge-test/pull/9) | **CONFLICTING** after #7 — Git still uses merge-base **C0**; squashed **S** is a new commit the branch history does not share |

This matches docs: squash “orphans” the branch; dependent PRs must rebase/merge `main`.

## Scenario C — distant lines (`loss-demo2.txt`, like Act 3)

| Step | PR | Result |
|------|-----|--------|
| [#10](https://github.com/Disturbing/merge-test/pull/10) | Squasher line 20 → `squasher=PRESENT` | Squash merged |
| [#11](https://github.com/Disturbing/merge-test/pull/11) | Your line 1 only (stale) | Squash merged, **CLEAN** |

**Final:** `line1=YOUR_PR` **and** `squasher=PRESENT` — **nothing lost.**

## Answers your question

> So it’s the person who squash merges after my PR merging in which loses their features, not mine?

**Partially.** Both orderings are risky:

| Order | Who is at risk | What we observed |
|-------|----------------|------------------|
| **They squash first, you merge stale** | **Their** squash work | Blocked on conflict (**#8**), or kept if edits are far apart (**#11**), or lost only if conflict resolved wrong (local demo) |
| **You merge first, they squash after** | **Their** squash can redo/duplicate/conflict with your merge | Not re-run here; same class of stale-base problem |

You are **not** guaranteed to keep your PR’s changes either — **#8** also blocks **you** until you update.

Git’s default is **correctness over convenience**: conflict or block, not silent merge that preserves both sides when the same region diverged.

## The right fix (both sides)

```bash
git fetch origin
git checkout your-pr-branch
git rebase origin/main   # or: git merge origin/main
git push --force-with-lease origin your-pr-branch
```

Branch protection **“Require branches to be up to date”** exists to force this before merge.

## Open PRs

- [#8](https://github.com/Disturbing/merge-test/pull/8) — conflicting (Scenario A)
- [#9](https://github.com/Disturbing/merge-test/pull/9) — conflicting (Scenario B)
