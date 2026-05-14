# Sharding boards above 1024 seats

UGM v2.3 caps a single atomic `createGrid` at
`MAX_SINGLE_TX_CREATE_TOTAL_SEATS = 1024` seats and a single grid
at `MAX_TOTAL_SEATS = 4096` seats. There are two scaling paths
above 1024:

| Path | Seat range | Layer |
| --- | --- | --- |
| **A — Chunked single-grid creation** | 1025 – 4096 | On-chain |
| **B — SDK-side multi-grid sharding** | 4097+ | Off-chain (one logical board ↔ many gridIds) |

You can combine them: a 12,000-seat board is typically four 3,000-
seat shards, each created via path A.

## Path A — chunked single-grid creation (1025–4096 seats)

UGM splits creation into a shell phase and an `initialPrices`
backfill phase:

1. **`createGridShell(CreateGridShellParams)`** — packs the grid
   config (`taxToken`, `yieldToken`, `taxRateBps`,
   `forfeitureDuration`, `totalSeats`) and reserves the gridId.
   No seat economics yet.
2. **`appendInitialPricingChunk(gridId, uint128[] prices)`** —
   one or more calls. Each chunk continues writing into
   `_packedInitialPrices` from
   `_initialPricesSeatsWritten[gridId]` to
   `_initialPricesSeatsWritten[gridId] + prices.length`. UGM
   reverts with `TooManyPrices` if a chunk would exceed
   `totalSeats`.
3. **Done state.** When
   `_initialPricesSeatsWritten[gridId] == totalSeats`, UGM
   emits `GridInitialPricingComplete(gridId)`. Seat economics
   unlock.

While pricing is partial:

- `addBatch`, `claim`, `setPrice`, `abandonBatch`, etc. all
  revert with `InitialPricingIncomplete()`.
- `gridConfigV22(gridId)` returns the resolved fields, so the
  indexer / UI can render the shell while the rest of the
  prices land.

### Recipe

```ts
import { encodeFunctionData, parseUnits } from 'viem';
import { base } from '@takeover/sdk/networks';
import { walletClient, publicClient } from './clients';

const ugm = base.ugmV23;
const TOTAL_SEATS = 3000;
const CHUNK_SIZE = 1024; // safe under the 1024 atomic cap

const shell = {
  taxRateBps: 500n,
  totalSeats: BigInt(TOTAL_SEATS),
  forfeitureDuration: 0,
  taxToken: base.usdcAddress,
  yieldToken: '0x0000000000000000000000000000000000000000',
};

// 1. Shell creation.
const { request: shellReq } = await publicClient.simulateContract({
  abi: ugmAbi,
  address: ugm,
  functionName: 'createGridShell',
  args: [shell],
});
const shellHash = await walletClient.writeContract(shellReq);
const shellRcpt = await publicClient.waitForTransactionReceipt({ hash: shellHash });
const gridId = decodeGridCreated(shellRcpt); // see boards/creating-a-grid.md

// 2. N pricing chunks.
const allPrices: bigint[] = Array.from({ length: TOTAL_SEATS }, () =>
  parseUnits('5', 6) as unknown as bigint,
);

for (let written = 0; written < TOTAL_SEATS; written += CHUNK_SIZE) {
  const chunk = allPrices.slice(written, written + CHUNK_SIZE);
  const { request } = await publicClient.simulateContract({
    abi: ugmAbi,
    address: ugm,
    functionName: 'appendInitialPricingChunk',
    args: [gridId, chunk],
  });
  await walletClient.writeContract(request);
}

// 3. Wait for `GridInitialPricingComplete(gridId)` before users
// interact. The indexer surfaces this; clients can also poll
// `_initialPricesSeatsWritten` if needed (it's a public mapping
// in the storage layout).
```

Reverts you should know:

- `TotalSeatsRequiresChunkedCreation` → `createGrid` was called
  with > 1024 seats. Switch to `createGridShell`.
- `InitialPricingIncomplete()` → seat-economic op tried before all
  chunks landed.
- `TooManyPrices` → a chunk would push `_initialPricesSeatsWritten`
  past `totalSeats`. Trim the chunk.
- `InitialPricesAlreadyComplete` → trying to append after the
  grid is already fully priced.

### Chunk-size guidance

- Bigger chunks = fewer txs, but each tx is heavier and risks
  the per-tx gas cap (~25M on Base under typical gas prices).
- 512 prices/chunk is a good safe default. 1024 works in
  practice but leaves no headroom for fee fluctuations.
- Sub-100 chunks are wasteful; the per-tx overhead dominates.

## Path B — SDK-side multi-grid sharding (4097+ seats)

For boards above the on-chain `MAX_TOTAL_SEATS = 4096` cap, sharding
moves to the SDK layer. There is no on-chain "logical board" —
each shard is a regular grid; the SDK stitches them into one
canvas via the `GridRef[]` config:

```ts
import { createTakeoverSDK, base } from '@takeover/sdk';

const sdk = createTakeoverSDK({
  ...base.toConfig({ apiKey: process.env.TAKEOVER_API_KEY! }),
  grids: [
    { ugmAddress: base.ugmV23, gridId: '101' }, // shard 0 — top-left   3000 seats
    { ugmAddress: base.ugmV23, gridId: '102' }, // shard 1 — top-right  3000 seats
    { ugmAddress: base.ugmV23, gridId: '103' }, // shard 2 — bot-left   3000 seats
    { ugmAddress: base.ugmV23, gridId: '104' }, // shard 3 — bot-right  3000 seats
  ],
});
```

What the SDK does for you:

- **Aggregated reads.** `sdk.getBoard()` joins the seat sets of
  every `GridRef` into a single canvas indexed by
  `(gridId, seatId)`. Tile lookups walk the canvas.
- **Per-shard writes.** Buy / sell / setPrice ops land on the
  specific gridId the seat belongs to. The SDK's `tileBuy`
  helper takes a `(gridId, seatId)` pair so no caller has to
  reason about shard membership.
- **Indexer alignment.** Each gridId has its own subscription
  in the indexer; the SDK's React hooks (`useTile`, `useBoard`)
  fan-out / fan-in across shards transparently.

What the SDK does NOT do:

- **Cross-shard atomicity.** A "buy two adjacent tiles that
  span shards" tx is two transactions, not one. Apps that need
  atomic cross-shard semantics MUST encode that themselves
  (e.g. via a multicall router that bundles the shard-specific
  calls).
- **Tax token mixing.** Every shard is a regular grid with its
  own `taxToken` / `yieldToken`. If your app expects uniform
  economics, replicate the same `taxToken` and `yieldToken`
  per shard at create time. Documenting the per-shard
  variance is acceptable but the SDK won't paper over it.

### When to pick A or B

| Want | Pick |
| --- | --- |
| Up to 4096 seats, single creator, single yieldToken. | Path A |
| Multiple yield buckets — different shards, different adapters. | Path B |
| Maps that conceptually grow over time. | Path B (start with one shard, add more later) |
| One unified UI canvas, single creator transaction model. | Path A if it fits; Path B otherwise |
| Independent governance per region. | Path B (each shard can have its own hook module) |

## Failure modes

- **Forgetting `GridInitialPricingComplete`.** If you start a
  campaign while `_initialPricesSeatsWritten[gridId] <
  totalSeats`, every claim reverts with
  `InitialPricingIncomplete()`. Wait for the event.
- **Mixing tax tokens across shards.** The SDK joins reads but
  doesn't reconcile economics. Document the variance and surface
  per-shard tax token in your UI.
- **Nondeterministic shard ordering.** Different `GridRef[]`
  orderings produce different `(shardIdx, seatIdx)` mappings in
  some apps. Pin the order in config.
- **Skipping fork tests.** A 12k-seat board with four shards
  has a real surface: cross-shard buy flows, per-shard pause
  state, etc. Fork-test the multi-shard happy path on Base
  Sepolia before mainnet launch.
