# Squash-merge stale PR experiment

## Setup

| Step | Branch | README |
|------|--------|--------|
| Initial `main` | — | `Hello` / `Coop` |
| PR1 `change-line-1` | squash → `main` | `Hi` / `Coop` |
| PR2 `change-line-2` (never updated after PR1) | stale head | `Hello` / `Peter` |

Merge methods: **squash only** (merge commit and rebase disabled).

## PR1 — merged

- https://github.com/Disturbing/merge-test/pull/1
- Squash-merged successfully.
- `main` after PR1:

```
Hi
Coop
```

## PR2 — blocked on GitHub

- https://github.com/Disturbing/merge-test/pull/2
- GitHub status: **mergeable: false**, **mergeable_state: dirty** (conflict).
- **Squash and merge** refused (including `--admin`): merge commit cannot be cleanly created.

### Why

Merge base is still the initial commit (`Hello` / `Coop`). PR2’s squash *delta* is only line 2 (`Coop` → `Peter`), but the PR branch file still has line 1 `Hello` while `main` has `Hi`. GitHub’s mergeability check sees a **content conflict** on `README.md` before squash can run.

Local reproduction:

```bash
git checkout main
git merge --squash origin/change-line-2
# → CONFLICT in README.md
```

### What squash *would* produce if GitHub allowed it

Per GitHub docs, squash commit = changes on the PR branch **since the merge base**, applied on current `main`:

- Delta `merge-base..change-line-2`: line 2 only (`Coop` → `Peter`)
- Applied to `main` (`Hi` / `Coop`) → **`Hi` / `Peter`**

Not `Hi` / `Coop` (Peter is in the PR delta and would apply).  
Not `Hello` / `Peter` unless line 1 were reverted (that would require a different failure mode).

## Thesis takeaway

Stale parallel branches + squash on the **same file**:

1. Sibling PR may become **unmergeable** until you **Update branch** or resolve conflicts — not silently dropped.
2. When squash *does* land a line-only delta on current `main`, expect **both** integrations (`Hi` + `Peter`), not loss of the PR hunk with “no conflicts.”

## Reproduce

```bash
git clone https://github.com/Disturbing/merge-test.git
cd merge-test
git log --oneline --graph --all
gh pr view 2
```

---

## Act 2 — two files (mergeable stale squash)

When each line lives in its own file, GitHub allows squash on the stale branch:

| PR | Branch | Change | Result |
|----|--------|--------|--------|
| [#3](https://github.com/Disturbing/merge-test/pull/3) | `act2-line1` | `line1.txt` Hello → Hi | squash merged |
| [#4](https://github.com/Disturbing/merge-test/pull/4) | `act2-line2` (not updated) | `line2.txt` Coop → Peter | squash merged |

Final `main`:

```
line1.txt → Hi
line2.txt → Peter
```

Equivalent to **`Hi` / `Peter`** in the one-file README — both edits land; the stale branch does **not** drop the PR2 hunk when there is no merge conflict.
