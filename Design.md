# Token Vault (`contracts/token-vault`) — Architecture & Implementation Design

> **Status:** Draft  
> **Scope:** Issues **#448**, **#452**, **#453**, **#454**  
> **Target Crate:** `contracts/token-vault` (to be scaffolded; not yet present on `main`)

---

## 1. Context & Repository Alignment

`contracts/token-vault` does not currently exist in `Perpetua-Contracts` (which presently contains `archival-probe`, `factory`, `governance`, `stream`, and `stream-nft`). This document specifies the architecture for `contracts/token-vault` so that its initial implementation satisfies all four issues while matching the repository's existing Soroban conventions (see `contracts/stream/Cargo.toml` and `contracts/stream/src/`):

- **Standalone Cargo crate:** `[workspace]` with `members = ["."]`, `edition = "2021"`, `soroban-sdk = "27.0.6"` (Protocol 27).
- **Release profile:** `opt-level = "z"`, `lto = "fat"`, `codegen-units = 1`, `panic = "abort"`, `strip = "symbols"`, `overflow-checks = true`.
- **Testing stack:** `soroban-sdk` with `testutils` + `proptest = "1"` for property/fuzz testing.
- **Module layout:**
  ```text
  contracts/token-vault/
  ├── Cargo.toml
  └── src/
      ├── lib.rs        # Contract trait & public entrypoints
      ├── error.rs      # Modular VaultError enum & observability helpers (#452)
      ├── auth.rs       # Granular RBAC roles & guards (#448)
      ├── storage.rs    # Typed DataKeys & automated TTL extension (#453)
      ├── types.rs      # Vault config, ClaimItem, BatchClaimResult, Role
      ├── events.rs     # Structured Soroban events
      └── test/         # Unit & proptest suites
  ```

---

## 2. Issue Designs

### 2.1 Modular Error Types & Custom Error Messages (`#452`)

**Goal:** Provide deterministic, client-observable error codes and structured diagnostic context without inflating WASM size.

- **Grouped `#[contracterror]` enum (`src/error.rs`):**
  Assign stable, non-overlapping numeric ranges so SDKs and indexers can categorize failures immediately:
  - `100–199` **Lifecycle & Configuration:** `AlreadyInitialized = 100`, `NotInitialized = 101`, `Paused = 102`, `InvalidConfig = 103`, `ZeroAddress = 104`
  - `200–299` **Authorization & RBAC:** `Unauthorized = 200`, `MissingRole = 201`, `CannotRevokeLastAdmin = 202`, `PendingAdminNotSet = 203`
  - `300–399` **Accounting & Arithmetic:** `ZeroAmount = 300`, `InsufficientBalance = 301`, `ArithmeticOverflow = 302`, `ArithmeticUnderflow = 303`, `CapExceeded = 304`
  - `400–499` **Batch & Cross-Contract:** `EmptyBatch = 400`, `BatchTooLarge = 401`, `InvalidClaimItem = 402`, `TokenTransferFailed = 403`
  - `500–599` **Storage & Archival:** `EntryNotFound = 500`, `InvalidTtlConfig = 501`
- **Client observability:**
  - Every `VaultError` variant carries concise doc comments used by Soroban contract spec generation (`soroban contract bindings`).
  - For batch or complex validation failures where a single `u32` code is ambiguous (e.g., which item in a batch failed), emit a lightweight `VaultErrorEvent` topic/data tuple or return a structured error code before aborting, keeping `#[contracterror]` `Copy + Eq + Debug` and WASM-compact.
- **Defensive arithmetic:**
  - All balance, fee, and batch-total calculations use `checked_add`, `checked_sub`, `checked_mul`, and `checked_div`, converting `None` directly into `VaultError::ArithmeticOverflow` / `VaultError::ArithmeticUnderflow`.

---

### 2.2 Granular Role-Based Authorization (`#448`)

**Goal:** Eliminate single-key privilege concentration and prevent unauthorized configuration updates.

- **Role model (`src/types.rs`, `src/auth.rs`):**
  ```rust
  #[contracttype]
  #[derive(Clone, Copy, Debug, Eq, PartialEq)]
  pub enum Role {
      Admin,        // Role assignment, two-step admin rotation, emergency recovery
      ConfigAdmin,  // Update vault parameters (caps, fee bps, TTL policy)
      Pauser,       // Pause / unpause deposits and claims
      Operator,     // Execute operational sweeps / batch settlements
      Keeper,       // Permissionless/delegated TTL maintenance
  }
  ```
- **Storage & guards:**
  - Store role memberships as `DataKey::Role(Role, Address) -> bool` in `persistent` (or `instance` for singleton roles) storage, with `DataKey::AdminCount -> u32` to prevent accidentally revoking the last `Admin`.
  - Provide `require_role(env, &caller, Role::...) -> Result<(), VaultError>` which calls `caller.require_auth()` and verifies membership, returning `VaultError::MissingRole` on mismatch.
  - **Two-step admin rotation:** `propose_admin` (requires `Admin`) writes `DataKey::PendingAdmin(Address)`; `accept_admin` requires auth from the pending address before swapping roles.
  - **Least-privilege mapping:**
    - `grant_role` / `revoke_role`: `Admin` only.
    - `update_config` / `set_ttl_policy`: `ConfigAdmin` only.
    - `pause` / `unpause`: `Pauser` only.
    - `settle_batch` (if operator-gated): `Operator` only.
    - `extend_ttl`: Permissionless (or `Keeper`-assisted) since extending TTL is strictly monotonic and safe.

---

### 2.3 Automated Entry Lifetime Extension (`#453`)

**Goal:** Guarantee archival resilience for long-lived vault state and user balances following Soroban Protocol 27 best practices.

- **Instance vs. Persistent policy (`src/storage.rs`):**
  - **Instance storage** (contract metadata, `VaultConfig`, `TtlPolicy`, `Paused`, `PendingAdmin`): extended automatically on every state-mutating entrypoint via `bump_instance(env)`.
  - **Persistent storage** (per-user `Balance(Address)`, `Role(Role, Address)`, `ClaimNonce(Address)`): extended automatically on **every read and write** using `extend_ttl(threshold, extend_to)`.
- **Dynamic `max_ttl()` clamping:**
  - Never hardcode network max TTL. Query `env.storage().max_ttl()` at runtime and clamp `extend_to = target_ttl.min(max_ttl)` and `threshold = threshold_ttl.min(extend_to)`, matching the pattern in `contracts/stream/src/storage.rs`.
- **Configurable `TtlPolicy`:**
  - Store `TtlPolicy { instance_threshold, instance_extend_to, persistent_threshold, persistent_extend_to }` with safe defaults (e.g., 30-day floor in ledgers) and validation (`threshold <= extend_to`, `extend_to > 0`).
- **Proactive keeper entrypoint:**
  - Expose `bump_vault_ttl(env, keys: Vec<DataKey>)` so off-chain keepers can refresh dormant user balances before archival without mutating balances.

---

### 2.4 Streamlining Cross-Contract Invocation Overhead in Batch Claims (`#454`)

**Goal:** Minimize Soroban CPU instructions, memory, and ledger footprint during high-frequency batch claims.

- **Current bottleneck in naive batch claims:**
  - Invoking `token::Client::new(&env, &token).transfer(&vault, &recipient, &amount)` inside a `for item in claims` loop triggers `N` cross-contract calls, `N` sub-invocation frames, and repeated storage reads of `VaultConfig` / `Paused` / `TokenAddress`.
- **Optimization strategy:**
  1. **Single-pass validation & aggregation:**
     - Read `VaultConfig` and `Paused` **once** before the loop.
     - Enforce `1 <= claims.len() <= MAX_BATCH_SIZE` (`VaultError::EmptyBatch` / `VaultError::BatchTooLarge`).
     - Validate each `ClaimItem` (`amount > 0`, valid auth/proof/entitlement) and accumulate payouts in an in-memory `Map<Address, i128>` (coalescing multiple claims for the same recipient) using `checked_add`.
  2. **Coalesced cross-contract transfers:**
     - Instantiate `token::Client` **once** outside the transfer loop.
     - Issue **at most one `token_client.transfer(...)` per unique recipient** (or a single `transfer` when claiming multiple items for a single caller), cutting cross-contract invocations from `N` down to `U` (`U <= N`, often `U == 1`).
  3. **Batched storage updates & TTL bumps:**
     - Update each affected user's persistent balance once after aggregating their items, extending TTL once per touched key rather than per claim item.
  4. **Atomic failure:**
     - Validate all items and compute all checked deltas before committing storage writes and external token transfers, ensuring no partial state corruption on error.

---

## 3. Verification & Test Plan

All tests will live in `contracts/token-vault/src/test/` and run via `cargo test` inside `contracts/token-vault`:

1. **RBAC unit tests (`#448`):**
   - Verify every config mutator rejects callers lacking the exact required `Role` with `VaultError::MissingRole`.
   - Verify `revoke_role` rejects removing the final `Admin` (`VaultError::CannotRevokeLastAdmin`).
   - Verify two-step `propose_admin` -> `accept_admin` flow and rejection of unauthorized `accept_admin`.
2. **Error & arithmetic tests (`#452`):**
   - Trigger and assert every `VaultError` variant; verify `i128::MAX` overflow and underflow paths return `ArithmeticOverflow` / `ArithmeticUnderflow` instead of panicking.
3. **Archival / TTL tests (`#453`):**
   - Advance `env.ledger().sequence()` past `threshold`, invoke reads/writes and `bump_vault_ttl`, and assert `get_ttl` on both `instance` and `persistent` entries is extended and clamped to `env.storage().max_ttl()`.
4. **Batch claim performance & fuzz tests (`#454`):**
   - Unit test verifying `N` claim items for `U` unique recipients execute `U` token transfers (e.g., 20 claims for 1 recipient -> 1 transfer) and match balances exactly.
   - `proptest` fuzzing over arbitrary `Vec<ClaimItem>` (varying duplicates, amounts, and failure injections) asserting fund conservation (`vault_balance_before - vault_balance_after == sum(claimed)`) and zero panics.

---

## 4. Open Questions Before Implementation

1. **Asset model:** Should `token-vault` hold a single fixed SAC/SEP-41 token per vault instance (configured at `initialize`), or support multiple tokens per vault?
2. **Claim authorization model:** Are batch claims caller-initiated (one user claiming multiple epochs/tranches) or operator-initiated (distributing to multiple recipients), or both?
3. **Batch cap:** What `MAX_BATCH_SIZE` (e.g., `50` or `100`) best fits Protocol 27's per-transaction footprint and instruction budget?
