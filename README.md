# PrizeJudgeCore

<p align="center">
  <strong>A contest prize the organizer cannot keep, move or hand to a favourite.</strong>
</p>

<p align="center">
  <a href="https://github.com/valentinzubok/PrizeJudgeCore/actions/workflows/ci.yml"><img src="https://github.com/valentinzubok/PrizeJudgeCore/actions/workflows/ci.yml/badge.svg" alt="CI" /></a>
  <img src="https://img.shields.io/badge/GenLayer-Studio%20Dev%2061997-fbbf24?style=flat-square" alt="Studio Dev" />
</p>

This repository holds the intelligent contract, its tests, its lint script and its deploy record,
and nothing else. The console that uses it is a separate project: [PrizeJudge](https://github.com/valentinzubok/PrizeJudge).

---

## The trust problem

A contest is only as fair as whoever picks the winner, and the organizer usually both holds the
prize and picks. They can quietly change what "best" means after seeing the entries, pay a
friend, or keep the money by declaring that nothing was good enough. Entrants can also edit their
page after the deadline.

**PrizeJudge takes all of that away from the organizer.** The prize is escrowed in native GEN when
the contest opens, the rubric is fixed, every entry page is frozen at the moment it is entered,
and the winner is the entry that validators agree on in separated rounds.

```
open_contest(contest_id, title, rubric, entries_close_in, judge_interval,
             rounds_required, max_entries)                              PAYABLE
      the attached value is the prize. Everything that decides the result is fixed here.
      with value attached it never reverts: a contest it cannot open is refused in the
      return value and the amount is credited to the sender

enter(contest_id, entry_url, note)      one entry per address, before entries close,
                                        never the organizer. Validators fetch the page and
                                        freeze it (SHA-256 of the whole page + a bounded
                                        digest). Judging reads only the frozen copy.

judge(contest_id)                       anyone; takes a contest id and nothing else
      refused before entries close and before a full judge_interval since the last round
      no entries  -> the prize goes back to the organizer, no model asked
      otherwise validators agree on ONE answer: the index of the entry that best meets
      the rubric, or "none"
        same answer as the standing proposal -> one more confirmation
        a different answer                   -> the count starts again from 1
      rounds_required agreeing rounds -> FINAL: the prize goes to that entrant's wallet
                                         (or back to the organizer if the answer is none)

expire(contest_id)                      anyone, after the committed judging deadline:
                                        an unresolved contest returns the prize
withdraw_credit()                       take back what open_contest refused to escrow
```

### Why it fails the way it does

A prize is paid once and cannot be clawed back, so one round is only a proposal and a pipeline
problem never looks like an answer: a malformed model answer, a model error or a consensus
failure **reverts the transaction**.

| Risk | What stops it |
|---|---|
| The organizer picks a friend | The organizer does not pick. `judge` takes a contest id and nothing else; the answer is what validators agree on. The organizer cannot enter. |
| The rubric is rewritten after the entries are in | It is fixed in `open_contest`; there is no setter, owner or admin. |
| The organizer keeps the prize | The prize leaves only to the winning entrant, or back to the organizer if validators agree no entry meets the rubric, if there were no entries, or if the contest expired unresolved. |
| An entrant improves the page after the deadline | Pages are frozen when entered. On chain, Bob's page was never re-read. |
| One lucky or wrong round decides | `rounds_required` agreeing rounds, each a full `judge_interval` after the previous one, elapsed time not a calendar window. A different answer restarts the count. |
| An entry tells the judge to pick it | Entries, notes, title and rubric are fenced as untrusted data, inner fences are neutralized, and instruction-like phrasing is flagged and stored. On chain Bob's page was flagged and lost. |
| `"0"`, `true`, `0.0` or a name counted as an answer | `literal_index()` accepts only a JSON integer inside the entry list; `null` means none. Anything else reverts. |
| A refused contest strands its prize | Value attached to a reverting call is not returned on this network, so `open_contest` never raises once value is attached; the amount is credited. |
| A payout that pays nobody | Wallets are paid by an external transfer, after the books are updated. |
| A contest nobody judges | `expire` returns the prize after the committed deadline. |

## Live

| | |
|---|---|
| Network | GenLayer Studio Dev / Studio Next — chain `61997` |
| Contract | [`0xa755846E9f9e9120E44ddbee7a99B217c1c0A7eD`](https://explorer-studio-dev.genlayer.com/address/0xa755846E9f9e9120E44ddbee7a99B217c1c0A7eD) |
| Deploy record | [`STUDIO_DEV_DEPLOY.md`](STUDIO_DEV_DEPLOY.md) — every transaction of the demo, including the refused ones |

## Lint

`sh scripts/lint_current_sdk.sh` runs the official GenVM linter (`genvm-lint lint`, `check` and
`typecheck`) and CI runs it on every push. Every refusal raises `gl.vm.UserError`; every method of
the contract class takes `self`. See [`STUDIO_DEV_DEPLOY.md`](STUDIO_DEV_DEPLOY.md#lint).

## Demo pages

Copies of the pages the on-chain demo read are in [`fixtures/`](fixtures/).

Three entry pages are files in this repository: one meets the rubric
([`entry-tilelight.html`](fixtures/entry-tilelight.html)), one has built nothing and
tries to instruct the judge ([`entry-mapdream.html`](fixtures/entry-mapdream.html)), one
misses part of the rubric ([`entry-quickmap.html`](fixtures/entry-quickmap.html)).

## Tests

```bash
pip install -r requirements-dev.txt
python3 -m pytest -q      # 60 tests
```

[`tests/test_prizejudge.py`](tests/test_prizejudge.py): agreeing rounds award the prize, a
different answer restarts the count, none and no entries return it; the organizer cannot enter
and has no power; one entry per address, the deadline and the cap; frozen pages are what the judge
reads; no round before entries close or inside the interval, across a midnight boundary and under
hammering; twelve malformed answers, a model error and a consensus failure revert; fencing and
flagged injection; expiry; nine refused openings credit everything; contests do not share money;
the contract balance equals escrow plus credits after every step of a random run.

## License

[MIT](LICENSE) © 2026 Valentyn Zubok
