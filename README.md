# Tracing SDK docs

Integration guides for clients of the tracing backend: what a client sends to
the anchoring routes, what it gets back, what lands on chain, and the webhooks
it receives.

## Plugins

- [Tickets](plugins/tickets.md): anchor a lottery draw on chain (ticket
  batches, the lock, its timestamp, and the records that belong to the draw).

## Reference

- [Webhooks](reference/webhooks.md): signed notifications when an anchor is
  mined or fails, and how to verify and answer them.
