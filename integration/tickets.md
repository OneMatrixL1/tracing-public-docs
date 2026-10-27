# Tickets plugin: integration guide

The `tickets` plugin anchors a lottery draw on chain. It covers the ticket
batches of the draw, the lock of the draw, the timestamp of that lock, and a few
records that belong to the draw (payout events, the auditor's reconciliation,
the list of sub-agents). Ticket content never reaches the backend. The client
sends only hashes and totals.

This guide covers what a client sends, what it gets back, what lands on chain,
and the webhooks it receives. The source of truth is
`apps/backend/plugins/tickets/plugin.yaml`. The machine-readable schemas are in
`apps/backend/api/openapi.yaml` (`BatchEventRequest`, `LockEventAccepted`, and
so on), and you can read them live with `GET /api/anchors/schema?eventType=<name>`.

## Contents

- [Overview](#overview)
- [Authentication and common rules](#authentication-and-common-rules)
- [The life of a draw](#the-life-of-a-draw)
- [`batch`: anchor the tickets of a draw](#batch-anchor-the-tickets-of-a-draw)
- [`lock`: lock a draw](#lock-lock-a-draw)
- [`lockts`: timestamp the lock](#lockts-timestamp-the-lock)
- [`events`, `recon`, `subagent`: hash records](#events-recon-subagent-hash-records)
- [Looking records up](#looking-records-up)
- [On-chain events](#on-chain-events)
- [Verifying a ticket](#verifying-a-ticket)
- [Webhooks](#webhooks)
- [Errors](#errors)

Reference pages:

- [Numbering ticket batches](../reference/batch-numbering.md): how to set
  `sequence` on each batch.
- [Webhooks](../reference/webhooks.md): signing, verification, the signed
  response, retries and setup.

## Overview

All six event types share the anchoring routes. You pick one with the
`eventType` query parameter:

| `eventType` | Purpose | Anchoring | `POST /api/anchors` answers | `POST /api/anchors/batch` |
|---|---|---|---|---|
| `batch` | One batch of tickets of a draw | Merkle tree per request | `200` once confirmed on chain | Not supported |
| `lock` | Lock a draw | One log; the server derives the key | `202` with `key` = drawRoot | Yes |
| `lockts` | RFC 3161 timestamp of the lock | One log; the server derives the key | `202` with `key` = drawRoot | Yes |
| `events` | Payout events of a draw (paid, referred, void) | One log per hash | `202` | Yes |
| `recon` | Auditor reconciliation against the Vietlott ledger | One log per hash | `202` | Yes |
| `subagent` | List of registered sub-agents | One log per hash | `202` | Yes |

## Authentication and common rules

- Authenticate as a client with `X-API-Key: <key>` or HTTP Basic Auth. Every
  client anchors into its own registry contract, and lookups only ever search
  that contract. `X-Admin-Key` cannot anchor.
- Bodies are JSON. **Undeclared fields are refused** (`400`).
- A `bytes32` value is 64 hex digits, with or without a `0x` prefix. It must
  not be all zeros. A `bytes` value is `0x`-prefixed hex.
- `uint64` values are JSON integers from 0 to 2^64-1. Above 2^53, send them
  from a JSON library that keeps large integers exact. JavaScript `Number`
  does not.
- `draw` must be a non-empty string. The plugin keys its state on the exact
  string, so use one stable spelling per draw.
- Before the first anchor, check that `GET /api/contract` reports
  `status: ready`.

## The life of a draw

```
 sales            close of sales                   after the draw
---------------   -----------------------------    -------------------------
batch  (n times)  lock --> drawRoot --> TSA -->    events / recon (any time)
                                        lockts
```

1. **Sell.** Send each batch of tickets with `eventType=batch`. The call
   returns once the batch's root is confirmed on chain.
2. **Lock.** When sales are closed and the last batch is confirmed, send
   `eventType=lock` with the draw. The backend derives the **drawRoot**: the
   hash of the block that holds the draw's last anchored batch. That block
   commits to everything anchored before it. The response returns drawRoot
   as `key`. Keep it.
3. **Timestamp.** Ask a timestamp authority (TSA) for an RFC 3161 token over
   `SHA-256(drawRoot)`, then send it with `eventType=lockts`.
4. **Records.** Anchor the hashes of payout events (`events`) and of the
   reconciliation (`recon`) for the draw whenever they exist. Anchor the
   sub-agent list (`subagent`) whenever it changes.

## `batch`: anchor the tickets of a draw

```
POST /api/anchors?eventType=batch
```

### Request: `BatchEventRequest`

| Field | Type | Required | Notes |
|---|---|---|---|
| `draw` | string | yes | Draw identifier, non-empty |
| `channel` | string | yes | Sales channel |
| `sequence` | uint64 | yes | Number of the batch's first ticket: see [Numbering ticket batches](../reference/batch-numbering.md) |
| `ticketHashes` | bytes32[] | yes | The leaves, in order: 1 to 100 000, non-zero, no repeats |
| `value` | uint64 | yes | Total value of the batch, in the unit you use |

```json
{
  "draw": "draw-1",
  "channel": "channel-1",
  "sequence": 0,
  "ticketHashes": [
    "0x5555555555555555555555555555555555555555555555555555555555555555",
    "0x6666666666666666666666666666666666666666666666666666666666666666"
  ],
  "value": 1
}
```

The ticket hashes go into the tree **exactly as given**, in the order given.
The backend never re-hashes a leaf. How a ticket is hashed is up to the
client, and anyone who verifies a ticket later must hash it the same way.
Prefix your leaf hash with a byte other than `0x01` and `0x02` (for example
`0x00`), so that a leaf can never be mistaken for an inner node or a batch
root.

### Numbering batches: `sequence`

`sequence` is the number of the batch's first ticket, and each batch starts
where the previous one ended. See [Numbering ticket batches](../reference/batch-numbering.md)
for the rules.

### What gets anchored

The tree follows RFC 9162 section 2.1 over keccak256:

- An inner node is `keccak256(0x01 || left || right)`.
- A node left without a partner at the end of a level moves up unchanged.
- The result is **`ticketRoot`**.

The root on chain wraps the batch totals around it:

```
batchRoot = keccak256( 0x02 || ticketRoot (32 bytes)
                            || count      (uint32, big-endian, 4 bytes)
                            || value      (uint64, big-endian, 8 bytes) )
```

`count` is the number of tickets in the batch (`len(ticketHashes)`).

### Response: `200 BatchEventResult`

The call waits until the transaction is confirmed.

```json
{
  "anchorId":   "1f6b0c1e-3a7d-5e2b-9c4f-8d2a6e1b7c30",
  "batchId":    42,
  "batchRoot":  "0x...",
  "ticketRoot": "0x...",
  "txHash":     "0x..."
}
```

`anchorId` names this batch, as it names every anchor: the webhook for this
batch carries it as `data.anchorId`. `batchId` is the id of the stored tree,
which the webhook also carries and the lookup accepts as `&batchId=`. A retry
of the same batch answers the same two.

**Retrying is safe.** Sending the identical ordered list again returns the
same tree and anchors nothing. On `504 CONFIRMATION_TIMEOUT` the batch is
stored and still being anchored, so send the same request again. Changing the
order or the content of `ticketHashes` makes a different batch.

## `lock`: lock a draw

```
POST /api/anchors?eventType=lock
```

### Request: `LockEventRequest`

```json
{ "draw": "draw-1" }
```

The client sends only the draw. The server finds the draw's last anchored
batch (the confirmed batch with the highest id). It takes the **hash of the
block that holds it** as `drawRoot`, records the lock, and anchors it.

### Response: `202 LockEventAccepted`

```json
{
  "blockchainStatus": "SUBMITTED",
  "contractAddress": "0x5FbDB2315678afecb367f032d93F642f64180aa3",
  "key": "0x<drawRoot>",
  "anchorId": "9b2f4c1e-7d3a-4e8b-a0c5-2f6d1e9b7a40"
}
```

`anchorId` names this submission. The webhook that announces its end carries the
same value as `data.anchorId`.

`202` means the node took the transaction. It has not been mined yet. Use the
lookup or the webhook to learn when it lands.

### Refusals (`409`)

| `error` | Meaning |
|---|---|
| `DRAW_HAS_NO_BATCH` | No batch of this draw has been confirmed yet |
| `BLOCK_HASH_UNKNOWN` | The draw's last batch was anchored before the backend recorded block hashes |

### Behaviour to know

- The lock is recorded as soon as the request is accepted, before the
  transaction is mined.
- Locking a draw again replaces its recorded drawRoot. If more batches were
  confirmed in between, the new drawRoot covers them. The earlier lock stays
  on chain.
- A batch confirmed after a lock is not covered by that lock. Lock only after
  your last batch has returned `200`.

## `lockts`: timestamp the lock

```
POST /api/anchors?eventType=lockts
```

### Request: `LocktsEventRequest`

| Field | Type | Required | Notes |
|---|---|---|---|
| `draw` | string | yes | A draw that has been locked |
| `signature` | bytes (`0x` hex) | yes | DER of an RFC 3161 `TimeStampToken`, or of the whole `TimeStampResp` |

```json
{ "draw": "draw-1", "signature": "0x3082..." }
```

### Getting the token

The token's message imprint must be **SHA-256 over the 32 raw bytes of
drawRoot** (not over its hex text). Ask the TSA to include its certificate,
because the backend verifies the signature with the certificate inside the
token. With OpenSSL:

```bash
DRAW_ROOT=0x...   # the `key` returned by lock
printf '%s' "${DRAW_ROOT#0x}" | xxd -r -p > drawroot.bin          # 32 bytes
openssl ts -query -data drawroot.bin -sha256 -cert -out req.tsq
curl -s -H 'Content-Type: application/timestamp-query' \
     --data-binary @req.tsq "$TSA_URL" -o resp.tsr
SIGNATURE=0x$(xxd -p resp.tsr | tr -d '\n')
```

### What the backend checks

The backend reads the drawRoot from its record of the draw's latest lock, then
checks that:

1. The token is well-formed DER: a `TimeStampToken`, or a `TimeStampResp` with
   status *granted* or *granted with modifications*.
2. The `TSTInfo` is version 1 and imprints `SHA-256(drawRoot)` with SHA-256.
3. There is exactly one signer, and its CMS signature verifies. SHA-256/384/512
   with RSA PKCS#1 v1.5 or ECDSA are accepted. The signed attributes must
   commit to the `TSTInfo`.
4. The signer certificate has a **critical** extended key usage of
   timestamping and nothing else, and was valid at the token's `genTime`.
5. If the operator configured trusted TSA roots (`params.tsaRootsPEM`, compiled
   into the build), the certificate chains to one of them. Without that
   setting, the token proves integrity and the drawRoot it covers, **not who
   issued it**.

Revocation and TSA policy are not checked. The token goes on chain whole, so
anyone can re-check it later.

### Response: `202 LocktsEventAccepted`

This has the same shape as `lock`: `key` is the drawRoot the token covers, and
`anchorId` names the submission.

### Refusals (`409`)

| `error` | Meaning |
|---|---|
| `DRAW_NOT_LOCKED` | The draw has no lock |
| `INVALID_TIMESTAMP` | The token failed a check. `message` says which one |

A lock timestamp is accepted for the latest recorded lock **whether or not
that lock has been mined**. Confirm the lock first (lookup or webhook) if you
need that ordering.

## `events`, `recon`, `subagent`: hash records

Each of these anchors one hash, which you compute over the record. The backend
never sees the record. The same hash may be anchored more than once.

```
POST /api/anchors?eventType=events     { "hash": "0x...", "draw": "draw-1" }
POST /api/anchors?eventType=recon      { "hash": "0x...", "draw": "draw-1" }
POST /api/anchors?eventType=subagent   { "hash": "0x..." }
```

| Field | Type | Required | Types |
|---|---|---|---|
| `hash` | bytes32, non-zero | yes | all three |
| `draw` | string | yes | `events`, `recon` |

The response is `202` with
`{ "blockchainStatus": "SUBMITTED", "contractAddress": "0x...", "anchorId": "<uuid>" }`.

What each record is (its fields and its serialisation before hashing) is
agreed between the sender and the verifier. The backend does not define it.
For `subagent`, the record has not been settled yet.

### Many records in one transaction

`events`, `recon`, `subagent`, `lock` and `lockts` also accept
`POST /api/anchors/batch?eventType=<name>` with a JSON array of 1 to 200 bodies.
Validation is all-or-nothing, and an error names the `index` of the bad item.
The `202` body adds `count`. For `lock` and `lockts`, `key` becomes `keys`, one
per item, in order. The whole call is one transaction, so it has **one**
`anchorId`.

```json
[
  { "hash": "0x1111...", "draw": "draw-1" },
  { "hash": "0x2222...", "draw": "draw-1" }
]
```

## Looking records up

```
GET /api/anchors?eventType=<name>&hash=<0x...64 hex>
```

| `eventType` | `hash` is | Needs an index endpoint |
|---|---|---|
| `batch` | a ticket hash (a leaf) | no |
| `lock`, `lockts` | the drawRoot | yes, `501 QUERY_NOT_AVAILABLE` without one |
| `events`, `recon`, `subagent` | the record hash | yes, `501 QUERY_NOT_AVAILABLE` without one |

### `batch`: `BatchEventQuery`

```json
{
  "hash": "0x5555...",
  "state": "anchored",
  "proofType": "merkle",
  "proof": ["0x6666..."],
  "leafIndex": 0,
  "treeSize": 2,
  "root": "0x<batchRoot>",
  "treeRoot": "0x<ticketRoot>",
  "batchId": 42,
  "meta": {
    "draw": "draw-1", "channel": "channel-1", "sequence": 0,
    "value": 1, "count": 2, "ticketRoot": "0x..."
  },
  "txHash": "0x...",
  "blockNumber": 4182,
  "event": { "abi": {...}, "signature": "Batched(...)", "topic0": "0x...", "address": "0x...", "values": {...} }
}
```

- `state` is one of `unknown` (never seen), `queued`, `pending`, `submitted`,
  `anchored`, `failed`.
- `proof` is the audit path from the leaf to `treeRoot`, leaf side first. It
  is empty until the leaf is in an anchored tree.
- A ticket hash may appear in more than one batch. By default you get the
  latest tree that holds it. Add `&batchId=<id>` to read a specific one.

### Other types: `LockEventQuery` and the rest

```json
{
  "hash": "0x...",
  "txHashes": ["0x..."],
  "proofType": "transactionHash",
  "proof": ["0x..."],
  "event": { "abi": {...}, "signature": "...", "topic0": "0x...", "address": "0x..." }
}
```

An empty `txHashes` means the hash has not been seen on chain (yet). `lock`
and `lockts` share the same key (the drawRoot), so `eventType` decides which
log you get.

## On-chain events

Each record is a log emitted by your registry contract. `topic0` is the
keccak256 of the signature. `GET /api/anchors/schema?eventType=<name>` returns
the ABI entry, the signature and `topic0`.

| `eventType` | Signature | Indexed topics (after topic0) | Data |
|---|---|---|---|
| `batch` | `Batched(bytes32,bytes32,string,string,uint64,uint64,uint64,uint64)` | `batchRoot` | `ticketRoot`, `draw`, `channel`, `sequence`, `count`, `value`, `treeSize` |
| `lock` | `Locked(bytes32,string)` | `drawRoot` | `draw` |
| `lockts` | `LockTS(bytes32,string,bytes)` | `drawRoot` | `draw`, `signature` (the token) |
| `events` | `EventsAnchored(bytes32,bytes32,string)` | `hash`, `drawHash` | `draw` |
| `recon` | `ReconAnchored(bytes32,bytes32,string)` | `hash`, `drawHash` | `draw` |
| `subagent` | `SubagentAnchored(bytes32)` | `hash` | none |

`drawHash` is `keccak256(utf8(draw))`. It lets an index list every `events` or
`recon` record of one draw: query the index (`/api/index/events`) for topic2 =
`drawHash`.

## Verifying a ticket

To prove that a ticket was sold in a batch that is on chain:

1. Hash the ticket the way the seller did to get the `leaf`.
2. Run `GET /api/anchors?eventType=batch&hash=<leaf>` to get `proof`,
   `leafIndex`, `treeSize`, `meta`, `txHash`.
3. Rebuild `ticketRoot` from the leaf and the proof.
4. Rebuild `batchRoot` from `ticketRoot`, `meta.count` and `meta.value`.
5. Read the transaction receipt from the chain yourself. Find the `Batched`
   log from the registry address whose topic1 equals `batchRoot` and whose
   data `ticketRoot` matches.

```js
import { keccak256, concat, toBeHex } from "ethers"; // ethers v6

const node = (l, r) => keccak256(concat(["0x01", l, r]));

// RFC 9162 section 2.1.3.2: the root the path leads to, or null if it does not fit.
function rootFromProof(leaf, index, size, path) {
  let fn = index, sn = size - 1, r = leaf;
  for (const p of path) {
    if (sn === 0) return null;
    if ((fn & 1) === 1 || fn === sn) {
      r = node(p, r);
      while ((fn & 1) === 0 && fn !== 0) { fn >>= 1; sn >>= 1; }
    } else {
      r = node(r, p);
    }
    fn >>= 1; sn >>= 1;
  }
  return sn === 0 ? r : null;
}

function batchRoot(ticketRoot, count, value) {
  return keccak256(concat(["0x02", ticketRoot, toBeHex(count, 4), toBeHex(BigInt(value), 8)]));
}
```

To prove the batch came before the lock, check that the `Batched` log's block
number is at or below the block whose hash is the drawRoot.

## Webhooks

When an operator has registered a webhook endpoint for your client, the
backend POSTs a signed JSON body when an anchor **ends**: `anchor.completed`
once it is mined and succeeded, or `anchor.failed`. Signing, verification,
the signed response, retries and setup are described in
[Webhooks](../reference/webhooks.md). This section only shows what the tickets types
put in `data`.

### Envelope

| Field | Type | Notes |
|---|---|---|
| `id` | uuid | Delivery id, same as `X-Webhook-Id`. Constant across retries: dedupe on it |
| `type` | `anchor.completed` \| `anchor.failed` | |
| `createdAt` | RFC 3339 UTC | |
| `data` | object | Below |

### `data` for `batch` (a Merkle tree)

| Field | Type | Notes |
|---|---|---|
| `eventType` | string | `batch` |
| `contractAddress` | string | Your registry |
| `status` | `CONFIRMED` \| `FAILED` | |
| `anchorId` | uuid | The `anchorId` the `200` answered |
| `batchId` | integer | The tree's id, same as `batchId` in the lookup |
| `root` | bytes32 | `batchRoot`, the value on chain |
| `treeRoot` | bytes32 | `ticketRoot` |
| `treeSize` | integer | Number of tickets |
| `meta` | object | `draw`, `channel`, `sequence`, `value`, `count`, `ticketRoot` |
| `txHash`, `blockNumber` | | When a transaction was mined |
| `reason` | string | On failure only |

The ticket hashes are never listed. Use the lookup to find them.

```json
{
  "id": "6f0c1e4e-5d0a-4a52-9a53-7a2d3c0f8f11",
  "type": "anchor.completed",
  "createdAt": "2026-10-19T10:00:00Z",
  "data": {
    "eventType": "batch",
    "contractAddress": "0x5FbDB2315678afecb367f032d93F642f64180aa3",
    "status": "CONFIRMED",
    "anchorId": "1f6b0c1e-3a7d-5e2b-9c4f-8d2a6e1b7c30",
    "batchId": 42,
    "root": "0x...",
    "treeRoot": "0x...",
    "treeSize": 2,
    "meta": {"draw": "draw-1", "channel": "channel-1", "sequence": 0, "value": 1, "count": 2, "ticketRoot": "0x..."},
    "txHash": "0x...",
    "blockNumber": 4182
  }
}
```

`batch` already answers synchronously, so this webhook is a second signal.
It is useful when the call ended in `504 CONFIRMATION_TIMEOUT` or the
connection dropped.

### `data` for `lock`, `lockts`, `events`, `recon`, `subagent` (direct anchors)

| Field | Type | Notes |
|---|---|---|
| `eventType` | string | The type name |
| `contractAddress` | string | Your registry |
| `status` | `CONFIRMED` \| `FAILED` | |
| `anchorId` | uuid | The `anchorId` the `202` answered |
| `keys` | bytes32[] | The log keys, in request order: the **drawRoot** for `lock` and `lockts`, the record **hash** for the others |
| `count` | integer | `len(keys)`. More than 1 for a `/api/anchors/batch` call |
| `txHash`, `blockNumber` | | When a transaction was mined |
| `reason` | string | On failure only |

```json
{
  "id": "...",
  "type": "anchor.completed",
  "createdAt": "2026-10-19T10:05:00Z",
  "data": {
    "eventType": "lock",
    "contractAddress": "0x...",
    "status": "CONFIRMED",
    "anchorId": "9b2f4c1e-7d3a-4e8b-a0c5-2f6d1e9b7a40",
    "keys": ["0x<drawRoot>"],
    "count": 1,
    "txHash": "0x...",
    "blockNumber": 4190
  }
}
```

```json
{
  "id": "...",
  "type": "anchor.failed",
  "createdAt": "...",
  "data": {
    "eventType": "events", "contractAddress": "0x...", "status": "FAILED",
    "anchorId": "...", "keys": ["0x..."], "count": 1, "reason": "reverted",
    "txHash": "0x...", "blockNumber": 4191
  }
}
```

### Matching a webhook to your request

Every anchoring answer, of every type, carries an `anchorId`, and the webhook
for that anchor repeats it as `data.anchorId`:

| `eventType` | Store from the answer | Match on in the webhook |
|---|---|---|
| `batch` | `anchorId` from the `200` | `data.anchorId` |
| `lock`, `lockts`, `events`, `recon`, `subagent` | `anchorId` from the `202` | `data.anchorId` |

1. When you send, store a pending row keyed by `anchorId`.
2. When a webhook arrives, verify it and drop repeats by `X-Webhook-Id`. Then
   look the row up by `data.anchorId`, and record `status`, `txHash`,
   `blockNumber` or `reason`.

Limits:

- If a `batch` call ends in `504 CONFIRMATION_TIMEOUT`, you have no
  `anchorId` yet. Send the same request again: it answers the same tree and
  its `anchorId`. Until then, match on `meta.draw`, `meta.channel` and
  `meta.sequence`.
- A direct anchor sent before your endpoint was set gets no webhook at all.
  Settle it with the lookup.
- Webhooks for submissions made before this release have no `anchorId`.

### `reason`

| `reason` | Meaning |
|---|---|
| `reverted` | Mined and reverted |
| `cancelled` | Cancelled before it was mined |
| `rejected` | The node never took it |
| `reorged_out` | Another transaction took its place |
| `not_confirmed` | A direct anchor was not seen on chain within `WEBHOOK_CONFIRM_TIMEOUT` |
| `failed` | A Merkle tree failed for another reason; an operator has been alerted |

A direct anchor is watched only if your endpoint was already set when you
submitted it. A tree uses whatever endpoint is set when it ends.

## Errors

Every error body is `{ "error": "<CODE>" }`, sometimes with `field`, `index`,
`message` or `batchId`.

| Status | `error` | When |
|---|---|---|
| 400 | `INVALID_REQUEST_BODY`, `INVALID_<FIELD>` | Bad JSON, an unknown field, or a bad field value (`field`, and `index` inside an array) |
| 400 | `EMPTY_LEAVES`, `BATCH_TOO_LARGE` | `ticketHashes` is empty or over 100 000 |
| 400 | `BATCH_NOT_SUPPORTED` | `batch` sent to `/api/anchors/batch` |
| 401 | `UNAUTHORIZED` | Missing or bad credentials |
| 404 | `UNKNOWN_EVENT_TYPE` | Misspelt `eventType` |
| 409 | `DUPLICATE_HASH` | A ticket hash repeated in one batch (`index` is the second position) |
| 409 | `DRAW_HAS_NO_BATCH`, `BLOCK_HASH_UNKNOWN`, `DRAW_NOT_LOCKED`, `INVALID_TIMESTAMP` | Plugin refusals, with `message` |
| 413 | `REQUEST_TOO_LARGE` | Body over the limit |
| 422 | `BATCH_REJECTED` | The plugin refused the request without a code; nothing was stored or sent |
| 501 | `QUERY_NOT_AVAILABLE` | Lookup of a direct type with no index endpoint |
| 502 | `ANCHOR_REVERTED` (with `batchId`), `ANCHORING_FAILED`, `RPC_ERROR`, `INDEX_ERROR` | Chain or index failure |
| 503 | `POOL_EXHAUSTED`, `POOL_NOT_READY`, `SUBMIT_FAILED`, `SERVER_UNAVAILABLE` | Retry later |
| 504 | `CONFIRMATION_TIMEOUT` | `batch` stored and still anchoring: resend the same request |
| 504 | `POOL_TIMEOUT` | Retry later |
