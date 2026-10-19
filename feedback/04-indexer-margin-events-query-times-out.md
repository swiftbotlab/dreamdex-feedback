# Indexer: the SDK's `MarginEvents` query (account + `kinds` filter, newest first) fails with "database query error" for an active account
Severity: major
Area: API | SDK

`client.getMarginEvents(account, { kinds: [...] })` sends this query (src/perp/history.ts:1032):

```graphql
query MarginEvents($where: MarginEvent_bool_exp!, $limit: Int, $offset: Int) {
  _sdkWatermark: chain_metadata { chain_id latest_processed_block }
  MarginEvent(where: $where, order_by: {timestamp: desc}, limit: $limit, offset: $offset) {
    id account kind pool amount granter timestamp txHash
  }
}
```

with `where = { account: {_eq: <addr>}, kind: {_in: [...]} }`. Against the Hideki indexer this request
fails 5 out of 5 times after ~10.5 s with `database query error` for an account with a large
MarginEvent history, so the documented portfolio-return recipe (`kinds: ["Deposit","Credited","Withdraw"]`)
cannot be run for exactly the accounts it is meant for. The same filter without `order_by`, or
ordered by `id`, completes (3–4.5 s), and the account alone (no `kinds`) returns in 0.5 s, which
points at a missing index covering (account, kind, timestamp) and a ~10 s statement timeout.
Two related shapes fail every time regardless of account: `MarginEvent(distinct_on: kind)` and
`MarginEvent_aggregate(where: {account: ...}) { aggregate { count } }`.

## Steps to reproduce
1. Pick an active account, e.g. `0x420691db9d2b5a79210fda48c177471386b6dfd1` (a market maker that
   produces many `Locked`/`Unlocked` rows).
2. POST the SDK's query to `https://hideki-dev.smk.somnia.host/v1/graphql`:

```
curl -s -m 60 -H 'content-type: application/json' https://hideki-dev.smk.somnia.host/v1/graphql --data '{"query":"query MarginEvents($where: MarginEvent_bool_exp!, $limit: Int, $offset: Int) { MarginEvent(where: $where, order_by: {timestamp: desc}, limit: $limit, offset: $offset) { id account kind pool amount granter timestamp txHash } }","variables":{"where":{"account":{"_eq":"0x420691db9d2b5a79210fda48c177471386b6dfd1"},"kind":{"_in":["Deposit","Credited","Withdraw"]}},"limit":50,"offset":0}}'
```

3. Repeat a few times; then vary the shape (no `order_by`; `order_by: {id: desc}`; no `kind`).
4. Also try `query { MarginEvent(distinct_on: kind) { kind } }` and
   `query { MarginEvent_aggregate(where:{account:{_eq:"0x4206…dfd1"}}) { aggregate { count } } }`.

## Expected
A page of at most 50 rows in well under the SDK's request timeout, like the other filters.

## Actual
```
{"errors":[{"message":"database query error","extensions":{"path":"$","code":"unexpected"}}]}
```

Timings measured 2026-10-19 09:20–09:30 UTC (each line one request):

| query shape | result |
|---|---|
| account + `kind _in [Deposit,Credited,Withdraw]` + `order_by timestamp desc` (SDK) | 10.76 s ERROR, 10.54 s ERROR, 10.70 s ERROR, 10.50 s ERROR, 10.71 s ERROR |
| same filter, `order_by {id: desc}` | 4.43 s ok |
| same filter, no `order_by` | 2.97 s ok |
| account only, `order_by timestamp desc` | 0.48 s ok |
| account + `kind _in [Locked]`, `order_by timestamp desc` | 0.70 s ok |
| quiet account `0x567476bd…3f45`, same SDK shape | 0.59 s ok |
| `MarginEvent(distinct_on: kind)` | ERROR (3/3) |
| `MarginEvent_aggregate(where account) count` | 0.52 s ERROR |

The SDK wraps the failure as an indexer error; nothing in the response says "timeout", so a caller
cannot tell a transient outage from this deterministic failure.

## Evidence
- Request/response above; `scripts/indexer_timing.sh` in this repo re-runs the table.
- SDK query text: `src/perp/history.ts` lines 1032–1037 (0.32.1).

## Suggested fix
Add a composite index on `MarginEvent (account, kind, timestamp DESC, id)` (or order the SDK query by
the primary key as `LiquidationsQuery` already does, see the comment at history.ts:1041), and return
a distinguishable error (`statement timeout`) instead of the generic `database query error`.

## Environment
- `@somnia-chain/markets-sdk` 0.32.1 (query text read from source; the request was sent with curl)
- Indexer `https://hideki-dev.smk.somnia.host/v1/graphql`, `chain_metadata.latest_processed_block` ≈ 27,50x,xxx (at chain head)
- Somnia Hideki testnet, chain id 50383
- Date: 2026-10-19, 09:20–09:30 UTC
- Wallet: n/a (read-only); account queried `0x420691db9d2b5a79210fda48c177471386b6dfd1`
