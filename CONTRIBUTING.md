# Contributing

Thanks for thinking about contributing. This repo is part of the ByteMe
Networks operations stack, so a few conventions keep things consistent across
the fleet.

## Before you open a PR

1. **Open an issue first** for anything beyond a typo or a one-line bug fix.
   It's easier to redirect direction in 30 seconds of discussion than after
   you've written 200 lines.
2. **Search existing issues** — odds are good it's already been raised,
   deferred, or rolled into a sibling repo.
3. **Match the existing style.** If the repo is Python, follow the style
   already in the tree (PEP-8, type hints where the surrounding code uses
   them). For RouterOS scripts, match the comment density and section
   headers already in place.

## Commit messages

Keep them short and present-tense:

```
add VLAN-10 carve-out to onboard-mikrotik.rsc
fix off-by-one in IPQuiz subnet calculator
docs: clarify dual-license precedence in README
```

No conventional-commits prefixes required, but `fix:` / `docs:` / `chore:`
prefixes are welcome when they help skimming.

## What we'll (probably) decline

- Bulk reformat-only PRs across an entire repo
- Dependency bumps that haven't been justified in the description
- New features that don't have a stated use case
- Changes to `LICENSE` — that's governed by the canon at
  `_licenses/CPOL.md` and is not a per-repo decision

## What we love

- A bug report with a minimal repro
- A PR that adds the smallest possible change to fix exactly one thing
- A note in the issue saying "I tried X and Y first" — it saves a round trip

## Questions

Open an issue tagged `question`, or email `aaron@bytemenetworks.com`.
