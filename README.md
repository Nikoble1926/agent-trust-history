# agent-trust-history

A **signed, daily, hash-chained** record of [ERC-8004](https://eips.ethereum.org/EIPS/eip-8004)
agent **trust scores**, published here as a third-party timestamp (each GitHub
commit time anchors the chain head). This is a **commit-reveal** log: only the
tamper-evident chain is public; the per-agent scores themselves stay private and
can be revealed later and verified against the commitments below.

## What is published here

`history_chain.jsonl` — one JSON object per day, append-only. Each entry:

| field | meaning |
|---|---|
| `seq`, `prev`, `h` | hash chain. `h = sha256(canonical(entry without h & signature))`, `prev` = previous entry's `h` (first = `GENESIS`) |
| `signed_by`, `signature`, `signer_scheme` | EIP-191 `personal_sign(h)` by the signer below |
| `chains[]` | per chain: `agents_sha256` = sha256 of the sorted per-agent score list (the **commitment** to that day's scores) + `n_scored`, `latest_block` |
| `snapshot_sha256` | sha256 of the full private sidecar (`snapshots/YYYY-MM-DD.json`) holding the actual per-agent scores |
| `ts_utc`, `date`, `totals` | timestamp + counts |

**Not published** (private; this is the commit-reveal part): the signing key, the
Python venv, and `snapshots/` (the real per-agent scores). They are committed to
by `snapshot_sha256` / `agents_sha256` and can be revealed later for verification.

## Signer

```
0x4538368cD3180B69238A4fAf8DdF2C45C6Bab12a
```

A dedicated key for this chain only (not reused from any other service).

## How to verify

For each line in `history_chain.jsonl`:

1. **Chain linkage** — check `entry["prev"]` equals the previous entry's `h`
   (first entry's `prev` is `GENESIS`).
2. **Hash** — recompute `sha256(canonical(entry without "h" and "signature"))`
   using compact, key-sorted JSON (`separators=(",",":")`, `sort_keys=True`,
   `ensure_ascii=False`) and check it equals `entry["h"]`.
3. **Signature** — EIP-191 recover the signer of `personal_sign(h)` and check it
   equals `signed_by` (which must equal the signer address above):

   ```python
   from eth_account import Account
   from eth_account.messages import encode_defunct
   rec = Account.recover_message(encode_defunct(text=entry["h"]),
                                 signature=entry["signature"])
   assert rec.lower() == entry["signed_by"].lower()
   ```

4. **Reveal (later)** — when a day's `snapshots/YYYY-MM-DD.json` is disclosed,
   check `sha256(file_bytes) == snapshot_sha256`, and rebuild `agents_sha256`
   from the sorted per-agent list to confirm the scores match what was committed
   on that date.

Scores are computed read-only with the Agent Trust Oracle's own methodology
(value_avg / volume / recency; min 3 distinct clients). Educational; not advice.
