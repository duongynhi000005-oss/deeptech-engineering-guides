# 5 Common Soundness Bugs in Solana Anchor & Rust Groth16 Zero-Knowledge Circuits (And How to Catch Them Automatically)

**Tags:** `rust`, `solana`, `cryptography`, `security`

When auditing Rust smart contracts on Solana (Anchor / raw BPF) and Zero-Knowledge circuits (Groth16 / Plonk / R1CS), the most catastrophic exploits rarely involve exotic mathematics. Instead, they stem from **subtle boundary mismatches between Rust host code, on-chain account deserializers, and prime-field constraint systems**.

Below are 5 high-severity vulnerability classes we repeatedly encounter in production audits and bug bounty hunts—along with an automated Python static scanner you can run in CI before deploying to mainnet.

> **Grab the Full Security Scanner & 50-Point Audit Checklist ($5):** Download the complete [**Rust, Solana & ZK-SNARK Security Audit Scanner + 50-Point Checklist Kit**](https://ko-fi.com/nhi741323) or our [**Crypto/Bounty FIFO/LIFO Tax OS ($5)**](https://ko-fi.com/s/6f070d3e68) on Ko-fi (also on [GitHub Releases](https://github.com/duongynhi000005-oss/rust-solana-zk-security-audit-kit/releases/tag/v1.0.0)).

---

## 1. `SOL-001` (CRITICAL): Privileged `AccountInfo` Without Signer Verification

In raw Solana BPF instructions or hybrid Anchor handlers that accept `AccountInfo<'info>` (or `UncheckedAccount<'info>`), checking that an account's public key matches `vault.authority` is **useless** unless you also verify `authority.is_signer`:

```rust
// ❌ VULNERABLE: Attacker passes the admin's pubkey as a read-only non-signer account!
pub fn update_fee(ctx: Context<UpdateFee>, new_fee_bps: u16) -> Result<()> {
    require_keys_eq!(ctx.accounts.authority.key(), ctx.accounts.config.admin);
    ctx.accounts.config.fee_bps = new_fee_bps;
    Ok(())
}
```

**Fix:** Always enforce `Signer<'info>` at the type level or explicitly check `if !authority.is_signer { return Err(ProgramError::MissingRequiredSignature); }`.

---

## 2. `SOL-002` (HIGH): Signed Integer `.abs()` Panic on `i64::MIN`

Perpetual futures DEXs, PnL engines, and liquidity pool rebalance instructions often represent funding deltas or price ticks as signed `i64` integers. Calling `.abs()` on `i64::MIN` (`-9,223,372,036,854,775,808`) triggers a two's-complement overflow panic in Rust (or wraps back to a negative number in release builds without overflow checks):

```rust
// ❌ VULNERABLE: Panics or wraps to negative when delta == i64::MIN
let abs_delta: u64 = pnl_delta.abs() as u64;

// ✅ SAFE: `.unsigned_abs()` returns u64 without overflow for all i64 values including i64::MIN
let abs_delta: u64 = pnl_delta.unsigned_abs();
```

---

## 3. `SOL-003` (HIGH): Non-Canonical PDA Bump via `create_program_address`

`Pubkey::find_program_address(seeds, program_id)` iterates bump seeds from `255` down to `0` and returns the **first (highest) valid off-curve bump**—known as the canonical bump.

If an instruction instead accepts a user-supplied `bump: u8` and calls `Pubkey::create_program_address(&[b"user_escrow", user.key.as_ref(), &[bump]], program_id)`, a single user can derive **multiple valid PDAs** for the same seed prefix using lower non-canonical bumps (e.g., `251`, `248`), bypassing per-user deposit caps or nullifier registries.

**Fix:** Always derive the canonical bump via `find_program_address` during account initialization, persist `account.bump = canonical_bump`, and verify against that stored bump on every subsequent instruction.

---

## 4. `ZK-001` (CRITICAL): Allocated-But-Unconstrained R1CS Witness Variables

In Arkworks, Bellman, or custom R1CS circuit synthesizers, allocating a witness variable (`cs.alloc_witness(...)` or `new_witness_variable(...)`) only places a value into the prover's assignment vector—it **does not add a mathematical constraint** to the R1CS matrices $(A, B, C)$:

```rust
// ❌ VULNERABLE: Witness is allocated, but never bound inside cs.enforce(...)!
// A malicious prover can substitute ANY arbitrary field element for `secret_nullifier`.
let secret_nullifier = cs.alloc_witness(|| Ok(expected_nullifier))?;
```

**Fix:** Every allocated witness variable must participate in a non-trivial linear combination inside `cs.enforce(|| "constraint_name", |lc| ..., |lc| ..., |lc| ...)`, and your unit tests must include a **malicious prover test** that mutates the witness assignment directly and asserts `!cs.is_satisfied()`.

---

## 5. `ZK-002` (HIGH): Non-Canonical BN254 Scalar Field Deserialization

The BN254 scalar field modulus $r \approx 2^{254}$ fits inside a 256-bit (32-byte) integer with headroom. If your verifier or bridge relayer deserializes a 32-byte public input or nullifier using `Fr::from_le_bytes_mod_order(bytes)` instead of rejecting inputs $\ge r$, an attacker can submit $x$ and $x + r \pmod{2^{256}}$ as **two distinct 32-byte on-chain nullifier keys** that reduce to the **exact same field element** inside the ZK circuit—enabling a double-spend withdrawal!

**Fix:** Enforce canonical field deserialization (`CanonicalDeserialize` or explicit big-integer `< Fr::MODULUS` validation) before accepting any public input or nullifier on-chain.

---

## Automating Detection in Python (`solana_zk_scanner.py`)

We packaged a zero-dependency static analyzer that scans `.rs` files for all 5 patterns (`SOL-001`, `SOL-002`, `SOL-003`, `ZK-001`, `ZK-002`) and outputs a Markdown/JSON audit report:

```python
from solana_zk_scanner import SolanaZkAuditScanner

scanner = SolanaZkAuditScanner()
findings = scanner.scan_source(open("vault.rs").read(), file_path="vault.rs")
print(scanner.to_markdown_report(findings))
```

### Get the Full Audit Kit & 50-Point Checklist
- **Ko-fi Shop ($5):** [Rust, Solana & ZK-SNARK Security Audit Scanner + 50-Point Checklist](https://ko-fi.com/nhi741323)
- **Crypto/Bounty Tax OS ($5):** [Deep-Tech Freelance, Bounty & Crypto FIFO/LIFO Tax OS](https://ko-fi.com/s/6f070d3e68)
- **GitHub Release `v1.0.0`:** [duongynhi000005-oss/rust-solana-zk-security-audit-kit](https://github.com/duongynhi000005-oss/rust-solana-zk-security-audit-kit)
