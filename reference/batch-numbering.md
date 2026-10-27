# Numbering ticket batches

How a client numbers the batches it sends to the `tickets` plugin, through the
`sequence` field of a [`batch` request](../integration/tickets.md#request-batcheventrequest).

`sequence` is the number of the **first ticket** in the batch. Tickets are
numbered from 0 within a draw and channel, in the order they appear in
`ticketHashes`, so a batch holds tickets `sequence` to
`sequence + count - 1`, and each batch starts where the previous one ended:

```
next sequence = previous sequence + previous count
```

For example, in one draw and channel:

| Batch | `sequence` | Tickets (`count`) | Ticket numbers |
|---|---|---|---|
| 1st | 0 | 123 | 0 to 122 |
| 2nd | 123 | 50 | 123 to 172 |
| 3rd | 173 | 200 | 173 to 372 |

A verifier can then check, from the `Batched` logs alone, that no batch of a
draw was left out: the batches of each channel, sorted by `sequence`, must
start at 0 and leave no gap or overlap. The lock is the point after which the
list is complete.

Rules for the sender:

- A retry of a batch (after a timeout or a dropped connection) sends the
  **same** `sequence` with the same tickets. It is the same batch, and it is
  anchored once.
- Do not take the next `sequence` until the previous batch has answered
  `200`. If a batch is never anchored, send its tickets again under the
  sequence it had, so the numbering keeps no hole.

The backend does not check the numbering today: it stores `sequence` as sent
and puts it on chain. Keeping it continuous is the sender's job, and a gap is
visible to anyone who reads the logs.
