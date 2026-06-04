# merge-test

This repo is a proof ledger for testing whether stale PRs lose data when GitHub
uses squash merges.

## Conclusion

For the clean separated-line case, **no data was lost**.

The final 10-line `README.md` experiment ended as:

```text
1  Hi
2
3
4
5
6
7
8
9
10 Peter
```

That means:

- PR 1's line 1 change (`Hello` -> `Hi`) was kept.
- PR 2's stale branch line 10 change (`Coop` -> `Peter`) was also applied.
- Squash merge did not copy the stale branch snapshot over `main`; it applied the
  PR delta from the merge base.

## PR Proof Map

| PR | Branch | Scenario | Outcome | Proof |
| --- | --- | --- | --- | --- |
| [#1](https://github.com/Disturbing/merge-test/pull/1) | `change-line-1` | Original two-line README, line 1 `Hello` -> `Hi` | Squash merged | `main` became `Hi` / `Coop` |
| [#2](https://github.com/Disturbing/merge-test/pull/2) | `change-line-2` | Original two-line README, stale line 2 `Coop` -> `Peter` | Blocked/conflicting, closed | Same-file adjacent change was not silently merged |
| [#3](https://github.com/Disturbing/merge-test/pull/3) | `act2-line1` | Two-file variant, line 1 file `Hello` -> `Hi` | Squash merged | First half of clean non-conflicting case |
| [#4](https://github.com/Disturbing/merge-test/pull/4) | `act2-line2` | Two-file variant, stale line 2 file `Coop` -> `Peter` | Squash merged | Final state kept both `Hi` and `Peter` |
| [#5](https://github.com/Disturbing/merge-test/pull/5) | `act3-line1` | 10-line README, line 1 `Hello` -> `Hi` | Squash merged | First half of same-file, far-apart case |
| [#6](https://github.com/Disturbing/merge-test/pull/6) | `act3-line2` | 10-line README, stale line 10 `Coop` -> `Peter` | Squash merged cleanly | Final README kept both line 1 `Hi` and line 10 `Peter` |
| [#7](https://github.com/Disturbing/merge-test/pull/7) | `act4-squasher` | Squasher feature lands first | Squash merged | `main` gained `squasher=PRESENT` in `loss-demo.txt` |
| [#8](https://github.com/Disturbing/merge-test/pull/8) | `act4-yours-stale` | Stale PR after squasher changed nearby lines | Blocked/conflicting, closed | GitHub did not silently drop squasher work |
| [#9](https://github.com/Disturbing/merge-test/pull/9) | `act4-yours-dependent` | Dependent branch carried squasher commits after squasher was squash-merged | Blocked/conflicting, closed | Shows the dependent-branch squash footgun: rebase/update required |
| [#10](https://github.com/Disturbing/merge-test/pull/10) | `act4c-squasher` | Distant-line squasher change in `loss-demo2.txt` | Squash merged | First half of distant-line case |
| [#11](https://github.com/Disturbing/merge-test/pull/11) | `act4c-yours` | Stale distant-line PR after #10 | Squash merged cleanly | Final state kept both `line1=YOUR_PR` and `squasher=PRESENT` |

## What This Proves

Squash merge does **not** automatically replace `main` with the stale PR branch's
tree.

In Git terms, GitHub applies the PR's delta from the merge base onto current
`main`. If the stale PR did not change a line relative to the merge base, that
line is not part of the squash patch.

So in the 10-line README test:

```text
merge base:  line 1 Hello, line 10 Coop
PR #5:       line 1 Hi,    line 10 Coop
PR #6:       line 1 Hello, line 10 Peter
final main:  line 1 Hi,    line 10 Peter
```

## Where Data Can Still Be Lost

Data can still be lost if a stale PR explicitly reverts another change, rewrites
the same file region, or a human resolves a conflict by taking the stale side.
That is a conflict-resolution or bad-patch problem, not automatic squash-merge
behavior.

The blocked PRs (#2, #8, #9) are evidence that GitHub often stops the merge
instead of silently overwriting when the same region diverges.
