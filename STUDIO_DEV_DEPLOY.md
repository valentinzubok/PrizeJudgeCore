# PrizeJudge — Studio Dev (chain 61997) deploy record

| | |
|---|---|
| **Network** | GenLayer Studio Dev / Studio Next — chain `61997` |
| **Contract** | [`0xAdf0E11eC8Db4E0B4315182Ac78879a671cEf66a`](https://explorer-studio-dev.genlayer.com/address/0xAdf0E11eC8Db4E0B4315182Ac78879a671cEf66a) |
| **Deploy tx** | [`0x34290b06…69dea1`](https://explorer-studio-dev.genlayer.com/tx/0x34290b064354149fd4845b996a6638f433e8b5ab2da809d923f0de359d69dea1) from `0x06721Ec34a405d7bbfeF398fdD320EDBEE9C976F`, which has no role in the contract |
| **Source** | [`contracts/PrizeJudge.py`](contracts/PrizeJudge.py) — runner `py-genlayer:5jycge4q8k23462jtb0b9fyey1s9qz928sz2nbrd9mg4sxqg2qng` |
| **Source sha256** | `4664df43f66f544f2392b91a8410a28531e3778caaf494bddfc743e3d090dc5b` |
| **Previous deployment** | `0xa755846E9f9e9120E44ddbee7a99B217c1c0A7eD` — [its run](docs/STUDIO_DEV_V1_RUN.md): three entries, an injection attempt flagged, two agreeing rounds, the prize awarded |

## What changed: the deadline splits time in two

The review found that at and after the judging deadline both `judge` and `expire` were accepted,
so transaction ordering could decide between a payout to the entrant and a refund to the
organizer. Now:

| | before the deadline | at or after the deadline |
|---|---|---|
| `judge` | accepted (after entries close, one interval apart) | **refused** before any fetch or model call: `the judging deadline has passed at …; only expire is accepted` |
| `expire` | **refused**: `the judging deadline has not passed` | accepted |
| `get_schedule` | `can_judge` may be true, `can_expire` false | `can_judge` false, `can_expire` true |

`judging_deadline = entries_close_at + (rounds_required + 3) × judge_interval`, fixed when the
contest opens. Tests in [`tests/test_prizejudge.py`](tests/test_prizejudge.py) (section "the
deadline"): one second before, exactly at, and after the deadline; an incomplete multi-round
settlement that cannot continue past it; three competing calls at the deadline reaching exactly
one outcome; a contest with no entries; and the schedule checked second by second around the
deadline, never offering both paths.

## Verify the source yourself

```bash
curl -s -X POST https://studio-dev.genlayer.com/api -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"gen_getContractCode","params":["0xAdf0E11eC8Db4E0B4315182Ac78879a671cEf66a"]}' \
  | python3 -c "import sys,json,base64,hashlib; print(hashlib.sha256(base64.b64decode(json.load(sys.stdin)['result'])).hexdigest())"
shasum -a 256 contracts/PrizeJudge.py
# both print 4664df43f66f544f2392b91a8410a28531e3778caaf494bddfc743e3d090dc5b
```

## Lint

`sh scripts/lint_current_sdk.sh` — `genvm-lint lint`, `check`, `typecheck`: passed, in CI. See the
previous record for why `check` runs on a header-swapped copy.

## The run: the deadline on chain

| Role | Address |
|---|---|
| organizer | `0xa244580D4742414b54C5860984aad2eA7877D15c` |
| Alice | `0x310bBE6f284497dA26F1b120eBd0b7C4befC474a` |

| # | Call | Sender | Result | What happened | Tx |
|---|------|--------|--------|---------------|----|
| 1 | `open_contest(late-1)` value 2 GEN | organizer | SUCCESS | `late-1`: 2 GEN, entries close in 60 s, 2 agreeing rounds 60 s apart. Judging deadline = close + (2 + 3) × 60 s = 1792510444 | [`0xb6222e33…084859`](https://explorer-studio-dev.genlayer.com/tx/0xb6222e33a5d38f551595864b3e1924315e8f4b880dc1a4909a02071f6d084859) |
| 2 | `open_contest(fast-1)` value 1 GEN | organizer | SUCCESS | `fast-1`: 1 GEN, 1 round | [`0x6306aed4…631628`](https://explorer-studio-dev.genlayer.com/tx/0x6306aed4e32a764b6b99a7300917cd48695f6e143b8996a0a2f1a605f1631628) |
| 3 | `enter(late-1)` | Alice | SUCCESS | Tilelight entry frozen in `late-1` | [`0x949fba37…a9a231`](https://explorer-studio-dev.genlayer.com/tx/0x949fba3794b3333d2cb4034d0c0e141c81bacd3b6803577d00ce68ad02a9a231) |
| 4 | `enter(fast-1)` | Alice | SUCCESS | Tilelight entry frozen in `fast-1` | [`0xb393ba1e…09892a`](https://explorer-studio-dev.genlayer.com/tx/0xb393ba1e692ce4d9af28e76c97448868361a73d08fbea242ecb3e8942409892a) |
| 5 | `judge(fast-1)` | stranger | SUCCESS | FINALIZED. `fast-1` judged well before its deadline: entry 0, **awarded** to Alice | [`0xce54f334…4a370c`](https://explorer-studio-dev.genlayer.com/tx/0xce54f334cbb39fe224ea61d141d673d90e123ebe69d64a6c02c8ecd42f4a370c) |
| 6 | `judge(late-1)` | stranger | SUCCESS | `late-1` round 1: entry 0 proposed, 1 of 2 | [`0x22ae965c…255e4b`](https://explorer-studio-dev.genlayer.com/tx/0x22ae965c84fac2566dc71c1bf32991a00e5b87651bd095b448e34e9773255e4b) |
| 7 | `expire(late-1)` | stranger | **ERROR** | **UserError** `the judging deadline has not passed; expiry opens at 1792510444` — before the deadline only `judge` is accepted | [`0x8872da72…b1e920`](https://explorer-studio-dev.genlayer.com/tx/0x8872da7295ced2e48afa572f85fc93ebc91dbc85b20fa429bed81d6c18b1e920) |
| 8 | `judge(late-1)` | stranger | **ERROR** | sent after the deadline (chain time 1792511088): **UserError** `the judging deadline has passed at 1792510444; only expire is accepted` — refused before any model call, although round 2 was still missing | [`0x3ab89701…144346`](https://explorer-studio-dev.genlayer.com/tx/0x3ab89701e2a9cf90e46d524fd6a7a3efe247ba97c317f227fdb7e01e4a144346) |
| 9 | `expire(late-1)` | stranger | SUCCESS | FINALIZED. `expire`: the prize returns to the organizer, status `expired` | [`0xded71760…3d3d1c`](https://explorer-studio-dev.genlayer.com/tx/0xded71760b27351c8dc85ce1f55e7b82e2b29932a9811f3c76e1f12e3f43d3d1c) |
| 10 | `judge(late-1)` | stranger | **ERROR** | **UserError** `the contest is settled` | [`0x8f8b5b08…118ab0`](https://explorer-studio-dev.genlayer.com/tx/0x8f8b5b088cd854ee4a06fff0514bbc973f1b0cdf951bce17b0a4c9c44d118ab0) |

`get_schedule("late-1")` read right after the deadline:
`{"can_judge":false,"can_expire":true,"confirmations":1,"expires_at":1792510444,"now":1792511088,"status":"judging"}`.

End state: `late-1` expired (prize back to the organizer, 1 of 2 rounds), `fast-1` awarded to
Alice, contract balance 0.
