# Raw runs

Unedited probe output, kept as-is.

- `2026-07-20-bradbury-deploy-timeout-THROW.log` — a run where the deploy timed
  out at the client and the RPC returned `fetch failed`. Kept deliberately: it
  shows the `THROW` verdict doing its job, and it is an honest sample of
  bradbury's variance. A tool that only ever ships its successful runs is not
  telling you what using the network is like.

Transaction hashes quoted in the README are on bradbury and can be queried
directly. We assert they correspond to the code in this repository; no third
party has attested that mapping.
