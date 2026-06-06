# Step 1 Judge: Voice/Video Calls Benchmark

Operator-only verdict from the clean run.

- Grep thread: `019e96b2-7923-7303-b460-38721aa6d89f`
- AST thread: `019e96b2-7df4-7973-8fc3-4cc23d3bdd31`
- Status: both completed from `/Users/defendend/workshop`
- Follow-up steering: none sent

Do not use this file as input to a new clean run.

## Verdict

Winner: `AST first, grep confirm`.

The win is not about speed. Grep-only finished faster and was strong at literal/details discovery. AST-first produced the better architecture map for future implementation planning.

## Why AST Wins

- Better structural coverage.
- Clearer ownership boundaries.
- Better separation of private calls, group calls, conference paths, native runtime, and OS integration.
- More reliable identification of state owners and runtime handoff.

## Why Grep Still Matters

`grep-only` is useful as a confirmation layer. It quickly finds literal details such as manifest declarations, plist keys, permission names, string keys, service names, and exact integration tails.

## Workshop Sound Bite

Use AST first to find ownership and flow. Use grep second to confirm literals and platform integration details.
