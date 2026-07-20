# genlayer-consensus-probe

Measure whether a GenLayer Intelligent Contract **actually reaches consensus**,
how often, and how it fails.

```bash
probe --contract ./my_contract.py --args '["BTC","base"]' --n 10
```

---

## Why this exists

A contract that returns the right value once has not been shown to reach
consensus. **Consensus is a distribution, not a single call.** The same contract
can return `{AGREE: 5}` on one run and split `3 AGREE / 2 DISAGREE` on the next
because a validator hit a slow endpoint, a CDN served different bytes, or an
upstream returned a 5xx to one validator and a 200 to another.

The GenLayer Studio tells you whether *a* transaction succeeded. It does not
tell you your contract's consensus *rate*, its failure *shape*, or its latency
distribution across repeated runs. That is what this measures.

It also encodes a set of receipt-reading caveats that otherwise each cost a
debugging session (see [Field notes](#field-notes)).

---

## What it reports

For each run it performs a full deploy + write cycle and reports **both stages
separately**:

- **A verdict** from an 8-way taxonomy:
  `AGREE_SUCCESS`, `AGREE_ERROR`, `DV`, `CANCELED`, `TIMEOUT`, `THROW`,
  `SKIPPED`, `OTHER`
- **The per-validator vote vector** (not just a boolean) — so you can see a
  4-1 squeaker differently from a clean 5-0
- **Whether all validator result hashes were identical**
- Contract address, elapsed time, and whether the time budget was hit

Across runs it aggregates verdict counts and **median / p95 latency** for
deploy, write, and total.

Output is machine-readable: one `RUN::{json}` line per run, then a final
`BATCH_SUMMARY::{json}`.

### The verdict taxonomy is the point

`AGREE_ERROR` is the one people miss. A transaction can be **ACCEPTED with all
validators agreeing** — and what they agreed on is that your contract raised.
Counting that as success is how a broken oracle looks healthy on a dashboard.

`DV` is decided by **vote-vector precedence**, not by the status label: if 4+
validators report `DISAGREE` or `DETERMINISTIC_VIOLATION`, it is a DV regardless
of what the top-level status says. A run can be labeled ACCEPTED and still be a
split underneath.

---

## Install and run

```bash
npm install
npx tsx src/probe.ts --help
```

For bradbury you need a funded test wallet. Put its `0x`-prefixed 32-byte
private key in a file and point at it:

```bash
export GENLAYER_WALLET_FILE=~/.cache/genlayer-test-wallet.txt
# or: --wallet /path/to/key
```

### Options

| Flag | Meaning |
|---|---|
| `--contract <path>` | **Required.** The `.py` Intelligent Contract to deploy. |
| `--args '<json>'` | Constructor args as a JSON array. Default `[]` |
| `--network <name>` | `bradbury` or `localnet`. Default `bradbury` |
| `--n <N>` | Independent deploy+write cycles, 1..50. Default `1` |
| `--budget <seconds>` | Per-run budget before giving up. Default `300` |
| `--method <name>` | Write method called after deploy. Default `resolve` |
| `--settle <seconds>` | Wait after deploy before the write. Default `15` |
| `--wallet <path>` | Key file (bradbury only) |

On `localnet` a fresh account is generated and funded automatically.

---

## Worked example

Running it against a no-LLM price oracle:

```
$ probe --contract ./price_oracle.py --args '["BTC","base"]' --n 1

[run 1] deployHash=0x1f015aa0c7b157d9e3f7c521f29af223a176a836cf4a5de69e18b37dee2f5d63
[run1_deploy]  poll status=REVEALING   exec=FINISHED_WITH_RETURN elapsed=6264ms
[run1_deploy]  poll status=ACCEPTED    exec=FINISHED_WITH_RETURN elapsed=12464ms
[run 1] resolveHash=0xb8edbb401f1c7ece0a3513d5bf191f364467e412926e130055357378bc9377ca
[run1_resolve] poll status=COMMITTING  exec=NOT_VOTED            elapsed=5380ms
[run1_resolve] poll status=ACCEPTED    exec=FINISHED_WITH_RETURN elapsed=10783ms

RUN::{"deployVerdict":"AGREE_SUCCESS","deployVotes":{"AGREE":5},
      "resolveVerdict":"AGREE_SUCCESS","resolveVotes":{"AGREE":5},
      "contractAddress":"0xCAaD875Dd5934c848A3d338a3900E1DEfCa4cE23", ...}
```

---

## Field notes

Every one of these cost a debugging session. They are baked into the probe so
they do not have to cost you one.

**1. Do not call a write method immediately after deploy.** It reverts at the
EVM level:

```
Transaction reverted: EVM tx 0xe19e3052... to consensus contract
0x0112Bf6e83497965A5fdD6Dad1E447a6E004271D was reverted.
```

Observed 2.2 seconds after a successful `{AGREE: 5}` deploy. The identical call
to the identical contract address succeeded once the deploy had settled. This is
why `--settle` defaults to 15 seconds. Set `--settle 0` to reproduce the revert.

**2. Use `getTransaction`, not `getTransactionReceipt`.** The latter is EVM-only
and reports "not found" for GenLayer transactions.

**3. Do not `waitForTransactionReceipt`.** The receipt indexer can lag well
behind finality. Submit, then poll `getTransaction`.

**4. `status: 14` is transient, not an error.** It appears mid-flight and
resolves on a later poll. Treat it as "keep polling".

**5. Votes are not where you would guess.** They live at
`tx.lastRound.validatorVotes` (numeric) and `tx.lastRound.validatorVotesName`
(labels), alongside `votesCommitted` / `votesRevealed`. The verdict is
`tx.resultName`; the execution outcome is `tx.txExecutionResultName`. There is
no `consensus_data.votes` on current bradbury responses.

**6. Statuses you will actually see while polling:** `COMMITTING`, `REVEALING`,
`ACCEPTED`, `FINALIZED`, `UNDETERMINED`, `CANCELED`. Only the last four are
decided.

---

## Limitations

- **Deploy-per-run by design.** Each run deploys a fresh contract so runs are
  independent. That costs testnet funds and time; it is the price of measuring a
  distribution rather than one contract's history.
- **Sequential only.** One wallet, one nonce stream. Parallelism would produce
  nonce collisions, not faster measurements.
- **It measures consensus, not correctness.** A contract can reach a clean 5/5
  on a wrong answer. Use it to find *instability*, not to validate logic.
- **Max N is 50** per invocation, deliberately, to bound spend.

---

## Provenance

This probe was extracted from the harness used to run a multi-phase study of
equivalence-principle behaviour on bradbury. The study it produced (an N=20
apples-to-apples comparison of an LLM-in-the-loop contract against a
deterministic one) and the contract used in the example above are separate
artifacts in their own repositories. Stating the relationship plainly:
**this is the instrument; the study and the contract are downstream of it.**
