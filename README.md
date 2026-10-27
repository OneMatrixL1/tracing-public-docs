# Tracing SDK docs

Integration guides for clients of the tracing backend: what a client sends to
the anchoring routes, what it gets back, what lands on chain, and the webhooks
it receives.

## [Tickets](integration/tickets.md): 
  anchor a lottery draw on chain (ticket batches, the lock, its timestamp, and the records that belong to the draw).

### [Numbering ticket batches](reference/batch-numbering.md): 
  how to set `sequence` on each ticket batch so a verifier can tell that no batch of a draw is missing.

### [Webhooks](reference/webhooks.md): 
  signed notifications when an anchor is mined or fails, and how to verify and answer them.

## [Verification portal](integration/portal.md): 
  the static page where users check an anchored ticket or record against the chain, and what a client provides for it (proposal).

### [Verification portal: integration guide](integration/portal-guide.md): 
  what a client builds so that its users can verify tickets on the portal: the App API, the verification link, the ticket format and the evidence file.

### 
