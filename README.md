# SecForge audit anchors

This repository is the **public verifiability layer** for the SecForge
ecosystem's hash-chained audit logs. Apps (Member Hub today; Proposal Forge
and Project Tracker as they ship) maintain tamper-evident `audit.event`
tables in their own databases. Every night, a CronJob in each app's
namespace:

1. Recomputes the chain from the last anchored row to the current latest.
2. Signs the latest row's `full_hash` with an Ed25519 key held in
   OpenBao Transit (`transit/keys/audit-signing`) — the private key
   never leaves the OpenBao cluster.
3. Commits a single JSON file into this repository under
   `<app-slug>/YYYY-MM-DD.json` recording:
   - the date,
   - the highest `audit.event.id` covered,
   - the row's hex `full_hash` (the chain root for that day),
   - the Ed25519 signature over that hash (base64),
   - the Transit key version + public key fingerprint at signing time.

**Tampering with an app's audit log now requires THREE simultaneous
compromises**:
1. Compromise the database superuser to mutate rows.
2. Compromise the OpenBao Transit private key to forge a matching
   signature for the rewritten chain.
3. Force-push to this public repository to overwrite the historical
   anchor — visible to anyone watching the repo via the GitHub Releases
   feed or a watch subscription.

## Per-app folders

```
member-hub/
  YYYY-MM-DD.json
proposal-forge/   # (future)
project-tracker/  # (future)
```

## Anchor file shape

```json
{
  "app": "member-hub",
  "date": "2026-05-20",
  "covers_through_id": 4178,
  "previous_anchor_id": 3992,
  "full_hash_hex": "f3a2b1...",
  "signature_b64": "MEUCIQ...==",
  "transit_key_version": 1,
  "transit_pubkey_b64": "BIMi2T/5x5IoGpLyQqx96+3wGz5AbjySTtm6QZu3o9w=",
  "schema_version": 1
}
```

## Verifying an anchor

`scripts/verify.py` (TBD) takes an anchor file + a copy of the app's
audit log dump and:
1. Recomputes the chain from the previous anchor's covered_through_id
   forward to this anchor's, asserting each row's `full_hash` matches.
2. Verifies the Ed25519 signature over the final `full_hash` using the
   pinned public key.
3. Cross-references the pinned public key against the live OpenBao
   Transit endpoint to detect a key swap.

Anyone — including non-operators — can run the verification.

## Why public

This repository is **deliberately public**. Public verifiability is the
threat model: any party with a stake in audit integrity (regulators,
tenants, partners, you-three-years-from-now) can verify the chain
against the commits here without needing access to SecForge
infrastructure.

## Anchor commit cadence

- Daily: midnight UTC.
- On-demand: an operator may trigger an extra anchor commit via the
  app's `audit-anchor` Job CR.
- Skip-and-resume tolerated: if the CronJob misses a day, the next run
  picks up from the last anchor's `covers_through_id` and anchors
  through "now", producing an anchor whose `date` field reflects the
  signing date.

## License

The audit anchors themselves are not copyrightable — they're hashes of
plaintext bytes. The scripts in this repo are Apache 2.0.
