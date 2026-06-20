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

## Methodology versioning & drift guard

Each entry from the foundation onward carries a `methodology` block **inside the
signed/hashed body**:

```json
"methodology": {
  "version": "v1.1",
  "params": { "weights": {...}, "min_distinct_clients": 3, "volume_ref": 50.0,
              "client_breadth_ref": 25.0, "recency_halflife_blocks": 50000 },
  "params_sha256": "sha256:...",
  "code_sha256": "sha256:...",
  "note": "coverage refinement, behavior unchanged"
}
```

- **`params_sha256` binds the methodology to the signed entry.** It is
  `sha256(canonical_json(normalised_params))` where canonical JSON uses
  `sort_keys=True, separators=(",",":"), ensure_ascii=False`, and **float
  normalisation** formats every float with Python `format(x, ".10g")` before
  serialising (so the hash never depends on float-repr quirks). The exact same
  function produces the hash on write and on the drift check.
- **`code_sha256` binds the scoring _logic_** (added in `v1.1`). It is
  `sha256` over the concatenated source of the score-affecting functions
  (`_normalise_value_to_100`, `_log_axis`, `_recency_weight`, `compute_score`),
  in fixed order. Where `params_sha256` pins the named constants, `code_sha256`
  pins the formula — a second tripwire so a change to the computation itself
  cannot slip through unversioned. Entries before `v1.1` simply lack this field.
- **Version boundaries (how a third party reads them):** scan entries by `seq`.
  All entries sharing a `methodology.version` were produced under the identical
  `params_sha256` **and** `code_sha256` — byte-for-byte the same scoring
  parameters and logic. A change in `version` marks an **intentional,
  human-acknowledged** methodology change. The `v1 -> v1.1` boundary is a
  **coverage refinement only**: `v1.1` records `client_breadth_ref` (previously
  an implicit constant) plus `code_sha256`; the scoring code is byte-for-byte
  unchanged, so scores are identical across it — flagged by the entry's
  `note: "coverage refinement, behavior unchanged"`. A future `-> v2` would
  signal a real change where scores before vs after are not directly comparable.
- **Drift guard.** The producer refuses to append if the live scoring params
  **or** the scoring code differ from the last recorded `params_sha256` /
  `code_sha256` **without** a matching version bump (silent drift) — it writes a
  private alert and stops instead. So within a single `version`, both the
  parameters and the scoring logic are guaranteed unchanged.
- **Retroactive declaration.** Entries written before this scheme existed (no
  `methodology` block) are covered by a one-time `declares` field on the first
  `v1` entry, stating which earlier `seq` range was produced under `v1`.

Verifiers should re-serialise **whatever fields each entry actually has** when
recomputing `h` (the schema is additive — older entries simply lack
`methodology`/`declares`).

## Snapshot files & reveal locating

Each chain entry records its own `snapshot_file`. **Always locate a snapshot via that field — never reconstruct the path from the date.** This lets both naming schemes coexist:

- **seq 0–2** (launch day 2026-06-20): date-keyed `snapshots/YYYY-MM-DD.json`.
- **seq 3 onward:** per-entry `snapshots/{date}_seq{N}.json` (collision-free, so same-day version boundaries each keep their own sidecar).

Snapshots stay private under commit-reveal; only each entry's `snapshot_sha256` digest is published here until a reveal.

### Launch-day reveal note (seq 0–2)

Three entries were written on 2026-06-20 under the old date-keyed scheme. seq 1 and seq 2 sidecars are byte-exact recoverable (sha matches the signed digest); **seq 0's exact bytes are not reproducible and were deliberately not reconstructed** — forging bytes to match a signed hash would defeat the commitment. Per-agent scores are identical across seq 0–2 (same `agents_sha256` in all three entries; `scoring.py` unchanged that day).

Scores are computed read-only with the Agent Trust Oracle's own methodology
(value_avg / volume / recency; min 3 distinct clients). Educational; not advice.
