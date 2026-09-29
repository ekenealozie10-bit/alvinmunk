# Canonical On-Chain Event Schema

> **Source of truth** for every Soroban event emitted by the Stellar Passport
> contracts. These shapes are **frozen** (belts/00-strategy §4) — changing a
> topic tuple or data layout breaks off-chain indexing. Add new fields by
> incrementing `schema_version`, never by reshuffling existing events.

## Legend

- **Topics** = the Soroban event topic vector (first element is the event
  discriminator; subsequent elements are indexed keys).
- **Data** = the non-indexed payload (a tuple/struct serialized via the
  contract's XDR encoding).
- &Version& = `schema_version` field if the event is versioned for forward
  compatibility.

---

## 1. Reputation Contract

### `att_set` (Attestation Set)

The **fundable primitive** (00-strategy §4). Emitted whenever an allowlisted
attester credits Earned XP. Versioned so future B2B consumers read the version
first and can evolve safely.

| Field | Type | Description |
|-------|------|-------------|
| **topics[0]** | `Symbol("att_set")` | Event discriminator |
| **topics[1]** | `Address` | The subject (who earned) |

**Data tuple** (schema_version = 1):

| Index | Type | Description |
|-------|------|-------------|
| 0 | `u32` | `schema_version` (currently `1`) |
| 1 | `Address` | `issuer` — the allowlisted attester contract/account |
| 2 | `u32` | `schema_id` — off-chain agreed namespace, passed through from `award_xp`. Every deployed quest uses `2` (QUEST). `1` is reserved and never emitted: vouches credit only the Social track, so they never produce `att_set` |
| 3 | `u64``| `amount` — XP credited by this award (a delta, not the running total; for the per-schema total read [`get_attestation`](#attestation)) |
| 4 | `u64``| `timestamp` — ledger timestamp at emission |

**Contract source**: `reputation/src/lib.rs` → `fn add_earned()`

```rust
// Emission (v1):
const ATTESTATION_SET: Symbol = symbol_short!("att_set");
const ATT_SCHEMA_VERSION: u32 = 1;
env.events().publish(
    (ATTESTATION_SET, to.clone()),
    (AATT_SCHEMA_VERSION, issuer.clone(), schema_id, amount, ts),
);
```

---

### `xp` (Earned Track Total)

Running total of the Earned (cashable) track for an address. A monotonic
sequence — indexers fold to get the latest balance per address.

| Field | Type | Description |
|-------|------|-------------|
| **topics[0]** | `Symbol("xp")` | Event discriminator |
| **topics[1]** | `Address` | The subject |

**Data tuple**:

| Index | Type | Description |
|-------|------|-------------|
| 0 | `u64` | `amount` — the delta just added |
| 1 | `u64``| `newTotal` — the new running total |

**Contract source**: `reputation/src/lib.rs` → `fn add_earned()`

```rust
env.events().publish(
    (symbol_short!("xp"), to.clone()),
    (amount, next),
);
```

---

### `social` (Social Track Total)

Running total of the Social (non-cashable, vouch-based) track. This is the
**leaderboard source**. Emitted on every social XP mutation (add or sub).
**Starter XP** (once-per-wallet `STARTER_SOCIAL = 20`) is **silent** — it
does NOT emit a `social` event, so brand-new wallets don't clutter the
event-sourced leaderboard until they first act.

| Field | Type | Description |
|-------|------|-------------|
| **topics[0]** | `Symbol("social")` | Event discriminator |
| **topics[1]** | `Address` | The subject (who gained/lost) |

**Data tuple**:

| Index | Type | Description |
|-------|------|-------------|
| 0 | `u64` | `amount` — an unsigned magnitude. The direction comes from comparing `newTotal` with the previous total (the address's prior `social` event, or the silent `STARTER_SOCIAL` balance for its first one): higher is a credit, lower is a debit |
| 1 | `u64` | `newTotal` — the new running total |

**Contract source**: `reputation/src/lib.rs` → `fn add_social()` / `fn sub_social()`

```rust
// Credit (add_social):
env.events().publish(
    (symbol_short!("social"), to.clone()),
    (amount, next),
);

// Debit (sub_social):
env.events().publish(
    (symbol_short!("social"), from.clone()),
    (amount, next),
);
```

---

### `attester` (Allowlist Change)

Emitted when an attester contract/account is added to or removed from the
allowlist.

| Field | Type | Description |
|-------|------|-------------|
| **topics[0]** | `Symbol("attester")` | Event discriminator |
| **topics[1]** | `Symbol("add")` or `Symbol("rm")` | Operation |

**Data**:

| Type | Description |
|------|-------------|
| `Address` | The attester address being added or removed |

**Contract source**: `reputation/src/lib.rs` → `fn add_attester()` / `fn remove_attester()`

```rust
// Add:
env.events().publish(
    (symbol_short!("attester"), symbol_short!("add")), attester);

// Remove:
env.events().publish(
    (symbol_short!("attester"), symbol_short!("rm")), attester);
```

---

### `vouch` (Async Half-Card Lifecycle)

Three sub-types track the lifecycle of an async vouch (the cold-start fix /
install funnel).

#### `vouch` / `minted`

A half-card is minted by `from` for an unknown recipient (bound to
`sha256(secret)`).

| Field | Type | Description |
|-------|------|-------------|
| **topics[0]** | `Symbol("vouch")` | Event discriminator |
| **topics[1]** | `Symbol("minted")` | Sub-type |

**Data tuple**:

| Index | Type | Description |
|-------|------|-------------|
| 0 | `u64``| `id` — auto-incremented vouch ID |
| 1 | `Address` | `from` — the voucher |

#### `vouch` / `claimed`

A recipient claims a half-card by presenting its secret.

| Field | Type | Description |
|-------|------|-------------|
| **topics[0]** | `Symbol("vouch")` | Event discriminator |
| **topics[1]** | `Symbol("claimed")` | Sub-type |

**Data tuple**:

| Index | Type | Description |
|-------|------|-------------|
| 0 | `u64``| `vouch_id` |
| 1 | `Address` | `from` — the original voucher |
| 2 | `Address` | `claimer` — the recipient who claimed |

#### `vouch` / `slashed`

An unclaimed half-card expires after its 7-day window; the staked Social XP
is forfeit (not refunded).

| Field | Type | Description |
|-------|------|-------------|
| **topics[0]** | `Symbol("vouch")` | Event discriminator |
| **topics[1]** | `Symbol("slashed")` | Sub-type |

**Data tuple**:

| Index | Type | Description |
|-------|------|-------------|
| 0 | `u64` | `vouch_id` |
| 1 | `Address` | `from` — the voucher whose stake was slashed |
| 2 | `u64` | `stake` — the slashed amount |

**Contract source**: `reputation/src/lib.rs` → `fn mint_vouch()` / `fn claim_vouch()` / `fn expire_vouch()`

```rust
// Mint:
env.events().publish(
    (symbol_short!("vouch"), symbol_short!("minted")), (id, from));

// Claim:
env.events().publish(
    (symbol_short!("vouch"), symbol_short!("claimed")),
    (vouch_id, vouch.from, claimer));

// Slash:
env.events().publish(
    (symbol_short!("vouch"), symbol_short!("slashed")),
    (vouch_id, vouch.from, vouch.stake));
```

---

## 2. QuestRegistry Contract

### `quest` / `created`

A new quest is registered by the admin.

| Field | Type | Description |
|-------|------|-------------|
| **topics[0]** | `Symbol("quest")` | Event discriminator |
| **topics[1]** | `Symbol("created")` | Sub-type |

**Data**:

| Type | Description |
|------|-------------|
| `u32` | `id` — the quest ID |

### `quest` / `awarded`

A quest is awarded to a recipient after off-chain attester verification.
Note: this event is emitted **after** the cross-contract call to
`Reputation.award_xp`, which itself emits `att_set` and `xp` events.

| Field | Type | Description |
|-------|------|-------------|
| **topics[0]** | `Symbol("quest")` | Event discriminator |
| **topics[1]** | `Symbol("awarded")` | Sub-type |

**Data tuple**:

| Index | Type | Description |
|-------|------|-------------|
| 0 | `u32` | `quest_id` |
| 1 | `Address` | `recipient` |

### `quest` / `attester_set` (Quest Attester Bind)

Emitted when the admin binds an attester key to a specific quest via
`set_quest_attester(quest_id, key)`. Once bound, only that key's signature
can authorize an award for the quest — the global attester allowlist is no
longer consulted for it. This is the boundary that keeps a partner's quest key
from acting as a treasury key. Monitoring (#116) should alert on this event.

| Field | Type | Description |
|-------|------|-------------|
| **topics[0]** | `Symbol("quest")` | Event discriminator |
| **topics[1]** | `Symbol("attester_set")` | Sub-type |

**Data tuple**:

| Index | Type | Description |
|-------|------|-------------|
| 0 | `u32` | `quest_id` — the quest the key is bound to |
| 1 | `BytesN<32>` | `key` — the attester public key now authorized for this quest |

**Contract source**: `quest_registry/src/lib.rs` → `fn set_quest_attester()`

```rust
env.events().publish(
    (symbol_short!("quest"), symbol_short!("attester_set")),
    (quest_id, key),
);
```

### `quest` / `attester_cleared`
(Quest Attester Unbind)

Emitted when the admin clears a quest's attester binding via
`clear_quest_attester(quest_id)`. After this the quest falls back to the
global attester allowlist. Monitoring (#116) should alert on this event.

| Field | Type | Description |
|-------|------|-------------|
| **topics[0]** | `Symbol("quest")` | Event discriminator |
| **topics[1]** | `Symbol("attester_cleared")` | Sub-type |

**Data**:

| Type | Description |
|------|-------------|
| `u32` | `quest_id` — the quest whose binding was cleared |

**Contract source**: `quest_registry/src/lib.rs` → `fn clear_quest_attester()`

```rust
env.events().publish(
    (symbol_short!("quest"), symbol_short!("attester_cleared")),
    quest_id,
);
```

### `streak` (Weekly Retention)

Emitted whenever a player's consecutive-week streak is updated (after a quest
award bumps it). A run that lapses without a new award emits nothing; see
[`Streak`](#streak) for how `get_streak` reports it.

| Field | Type | Description |
|-------|------|-------------|
| **topics[0]** | `Symbol("streak")` | Event discriminator |
| **topics[1]** | `Address` | `player` — the streak subject |

**Data tuple**:

| Index | Type | Description |
|-------|------|-------------|
| 0 | `u32` | `weeks` — the new consecutive-week count |
| 1 | `u32` | `best` — the all-time high |

**Contract source**: `quest_registry/src/lib.rs` → `fn bump_streak()`

```rust
env.events().publish(
    (symbol_short!("streak"), player.clone()), (s.weeks, s.best));
```

---

## 3. Registry Contract (Handles)

### `handle` / `claimed`

A wallet takes a handle, either its first one or as a rename. On a rename the
old handle is announced with `handle` / `released` in the same transaction,
Zimmediately before this event. Re-claiming the handle the wallet already holds
changes nothing and emits no event.

| Field | Type | Description |
|-------|------|-------------|
| **topics[0]** | `Symbol("handle")` | Event discriminator |
| **topics[1]** | `Symbol("claimed")` | Sub-type |

**Data tuple**:

| Index | Type | Description |
|-------|------|-------------|
| 0 | `Address` | `caller` — the claiming wallet |
| 1 | `Symbol` | `handle` — the claimed handle |

### `handle` / `released`

A handle is freed: the wallet released it (`release()`), or renamed away from
it (`claim()` with a different handle, emitted right before the new `claimed`).
Either way the handle no longer resolves and anyone may claim it.

| Field | Type | Description |
|-------|------|-------------|
| **topics[0]** | `Symbol("handle")` | Event discriminator |
| **topics[1]** | `Symbol("released")` | Sub-type |

**Data tuple**:

| Index | Type | Description |
|-------|------|-------------|
| 0 | `Address` | `caller` — the wallet that held the handle |
| 1 | `Symbol` | `handle` — the freed handle |

An indexer keyed by handle stays in sync by applying both sub-types in event
order: `claimed` sets `handle → caller`, `released` deletes `handle`. The one
gap is `admin_release()` (see the note below).

**Contract source**: `registry/src/lib.rs` → `fn claim()` / `fn release()`

```rust
// Rename (inside claim, before the claimed event):
env.events().publish(
    (symbol_short!("handle"), symbol_short!("released")),
    (caller.clone(), old));

// Claim:
env.events().publish(
    (symbol_short!("handle"), symbol_short!("claimed")),
    (caller, handle));

// Release:
env.events().publish(
    (symbol_short!("handle"), symbol_short!("released")),
    (caller, handle));
```

> **Note**: `admin_release()` does **not** emit a `handle` event (admin-only
> operation that cleans up state silently). It does emit `meta` / `cleared` when
> the holder had a profile.

### `meta` / `set`

A handle holder publishes its profile face and bio (`set_meta()`), replacing any
earlier ones. Only an address that holds a handle can set one; a rename keeps it.
The stored shape is [`ProfileMeta`](#profilemeta-get_meta).

| Field | Type | Description |
|-------|------|-------------|
| **topics[0]** | `Symbol("meta")` | Event discriminator |
| **topics[1]** | `Symbol("set")` | Sub-type |

**Data tuple**:

| Index | Type | Description |
|-------|------|-------------|
| 0 | `Address` | `caller` — the handle holder |
| 1 | `u64``| `avatar` — the packed face (layout under `ProfileMeta`) |
| 2 | `String` | `bio` — plain text, may be empty |

### `meta` / `cleared`

An address's profile is deleted because it gave up its handle: `release()`
(right after `handle` / `released`) or `admin_release()`. Emitted only when there
was a profile to delete. Meta is keyed by address, so whoever claims the freed
handle next starts with none.

| Field | Type | Description |
|-------|------|-------------|
| **topics[0]** | `Symbol("meta")` | Event discriminator |
| **topics[1]** | `Symbol("cleared")` | Sub-type |

**Data**:

| Type | Description |
|------|-------------|
| `Address` | The address whose profile was deleted |

An indexer keyed by address folds both in 
