# Security Audit: Subscription Smart Contract

**Auditor:** [Triumph Synergy Digital Financial Ecosystem](https://github.com/jdrains110-beep/Triumph-Synergy-Digital-Financial-Ecosystem)
**Contract:** `contracts/subscription/src/lib.rs`
**Date:** April 2026
**Scope:** Full review of the subscription contract Rust source code and architecture

---

## Executive Summary

The Pi Network Subscription Smart Contract is a well-engineered Soroban contract implementing merchant-initiated pull-payment subscriptions. This audit reviews the `lib.rs` source for security vulnerabilities, logic errors, and best-practice adherence. Overall, the contract demonstrates strong security fundamentals including proper auth checks, overflow protection, and failure isolation. We identify several areas for hardening and a few informational findings.

---

## Findings

### [S-01] No `deactivate_service` Method — Severity: Medium

**Location:** `SubscriptionContract` impl block

**Description:** The `Service` struct includes an `is_active` field, and `subscribe()` correctly checks `!service.is_active` before allowing new subscriptions. However, there is no public method for a merchant to set `is_active = false`. Once a service is registered, it accepts subscribers indefinitely.

**Impact:** Merchants cannot sunset a service or stop accepting new subscribers without an admin contract upgrade.

**Recommendation:** Add a `deactivate_service(merchant, service_id)` method that sets `is_active = false` after verifying `merchant.require_auth()` and service ownership. Existing subscriptions should continue to process normally.

---

### [S-02] Unbounded `Vec` Growth in Index Keys — Severity: Medium

**Location:** `register_service()`, `subscribe()`

**Description:** `MerchantServices`, `SubscriberSubs`, and `ServiceSubs` are `Vec<u64>` that grow unboundedly. Each new service/subscription appends to these vectors. Expired subscription IDs are never pruned from the vectors (only gracefully skipped during reads).

**Impact:**
- Over time, these vectors may grow large enough to exceed Soroban's per-entry storage size limits or make read operations expensive.
- `get_subscriber_subs` and `get_merchant_subs` iterate the full vector, loading every entry.

**Recommendation:** Consider either:
1. A lazy cleanup strategy — remove expired IDs during `process()` or query calls.
2. A paginated index structure to avoid unbounded single-entry growth.
3. Document the practical subscription limit per service/subscriber for integrators.

---

### [S-03] `next_service_id` / `next_sub_id` Increment Without Overflow Check — Severity: Low

**Location:** `next_service_id()`, `next_sub_id()` helper functions

**Description:**
```rust
env.storage().instance().set(&DataKey::NextServiceId, &(id + 1));
```

The ID counters use `id + 1` without `checked_add`. While `u64::MAX` (~18.4 quintillion) is practically unreachable, the contract uses `checked_add` elsewhere (`checked_add_ts`) demonstrating awareness of overflow.

**Recommendation:** Use `id.checked_add(1).expect("ID overflow")` for consistency with the contract's overflow-safe design philosophy.

---

### [S-04] `do_approve` Rounds Expiration Down — May Shorten Approval — Severity: Low

**Location:** `do_approve()` helper

**Description:**
```rust
let expiration_ledger = (capped / LEDGER_BUCKET) * LEDGER_BUCKET;
```

Rounding down to 720-ledger buckets ensures simulate/execute consistency, but can reduce the effective approval duration by up to 719 ledgers (~60 minutes). For short billing periods (e.g., hourly), this could represent a meaningful fraction of the approval window.

**Recommendation:** Consider rounding **up** instead of down: `((capped + LEDGER_BUCKET - 1) / LEDGER_BUCKET) * LEDGER_BUCKET` (capped at `max_expiration`). This ensures the approval is never shorter than intended.

---

### [S-05] `process()` Does Not Bump `pair_key` TTL — Severity: Low

**Location:** `process()` function

**Description:** During batch processing, `process()` bumps the `Sub(sub_id)` TTL on every charge but does not bump the `SubServicePair` TTL. If a subscription is active for a very long time without the subscriber calling any lifecycle method (`cancel`, `toggle_auto_renew`, `extend_subscription`), the `SubServicePair` entry could expire before the subscription itself.

**Impact:** After `SubServicePair` expires, the deduplication check in `subscribe()` would allow a second active subscription to the same service.

**Recommendation:** Bump `SubServicePair` TTL in `process()` alongside `Sub(sub_id)`, or document that subscribers must call a lifecycle method periodically.

---

### [S-06] Single Admin Key for `upgrade` — Severity: Informational

**Location:** `upgrade()`

**Description:** The `upgrade` function is gated by a single admin address. There is no key rotation mechanism, time lock, or multi-sig pattern.

**Impact:** Loss of the admin key permanently prevents upgrades. Compromise allows immediate malicious WASM replacement.

**Recommendation:** For mainnet, consider:
1. An `update_admin` method with dual-auth (old admin + new admin).
2. A time-lock pattern giving the community a window to react before upgrades take effect.

---

### [S-07] `is_subscription_active` Does Not Bump TTLs — Severity: Informational

**Location:** `is_subscription_active()`

**Description:** This is a read-only function that accesses `SubServicePair` and `Sub` persistent storage without bumping TTLs. This is by design (query functions use `simulateTransaction`), but integrators relying solely on this function without calling mutating methods may see data expire.

**Recommendation:** Document that `is_subscription_active` does not extend storage TTLs and that active subscriptions require periodic `process()` calls to maintain storage liveness.

---

## Positive Findings

These design choices demonstrate strong security engineering:

| # | Finding | Location |
|---|---------|----------|
| P-01 | **Proper `require_auth()`** on all state-changing functions | All mutating methods |
| P-02 | **`checked_add` for timestamps** prevents overflow panics | `checked_add_ts()`, `do_approve()` |
| P-03 | **`checked_mul` for approval amounts** prevents integer overflow | `do_approve()` |
| P-04 | **Failure isolation** — failed charges disable individual subscriptions, not the batch | `process()` |
| P-05 | **No-drift billing** — `next_charge_ts` advances from previous value, not `now` | `process()` |
| P-06 | **Trial abuse prevention** — prevents infinite free trial reuse | `subscribe()` dedup logic |
| P-07 | **Dynamic TTL** — persistent storage TTL scales with billing period | `ttl_extend_for_period()` |
| P-08 | **`saturating_mul` / `saturating_add`** for TTL calculations prevents panic | `do_approve()`, `ttl_extend_for_period()` |
| P-09 | **Deduplication via `SubServicePair`** prevents duplicate active subscriptions | `subscribe()` |
| P-10 | **Pre-authorized approve before transfer** in non-trial path | `subscribe()` |

---

## Summary Table

| ID | Title | Severity | Status |
|----|-------|----------|--------|
| S-01 | No `deactivate_service` method | Medium | Open |
| S-02 | Unbounded `Vec` growth in index keys | Medium | Open |
| S-03 | ID counters lack `checked_add` | Low | Open |
| S-04 | Approval expiration rounds down | Low | Open |
| S-05 | `process()` does not bump `SubServicePair` TTL | Low | Open |
| S-06 | Single admin key for `upgrade` | Informational | Open |
| S-07 | `is_subscription_active` TTL behavior | Informational | Open |

---

## Disclaimer

This audit is provided as a community contribution and does not constitute a formal security guarantee. It is based on source code review as of the audit date. The findings and recommendations are intended to help improve the contract's security posture.