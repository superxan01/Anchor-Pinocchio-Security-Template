# Solana Security Patterns: A Comprehensive Reference

**An educational repository for understanding common Solana vulnerabilities and secure patterns using the Anchor framework.**

> *This repository was created as a reference implementation for the Superteam Bounty on Solana Security. It serves as both a teaching tool and a practical guide for developers building secure programs on the Solana blockchain.*

---

## Table of Contents

1. [Introduction](#introduction)
2. [Why Solana Security is Hard](#why-solana-security-is-hard)
3. [Vulnerability Index](#vulnerability-index)
4. [Repository Structure](#repository-structure)
5. [Quick Start](#quick-start)
6. [How to Run](#how-to-run)
7. [Key Takeaways](#key-takeaways)
8. [Learning Path](#learning-path)
9. [Resources](#resources)

---

## Introduction

Welcome to **Solana Security Patterns**, a hands-on educational repository that explores the most critical vulnerabilities in Solana programs and demonstrates how to mitigate them.

### Who This Is For

- **Solana developers** learning security best practices
- **Auditors** needing concrete examples of vulnerabilities
- **Security researchers** studying blockchain attack vectors
- **Developers from other chains** new to Solana's unique security model
- **Teams preparing security reviews** of their own programs

### What You'll Learn

This repository demonstrates:

1. **5 critical vulnerability categories** found in real Solana exploits
2. **Insecure implementations** showing exactly what goes wrong
3. **Secure implementations** using Anchor best practices
4. **Detailed explanations** of why each vulnerability matters
5. **Gas/performance trade-offs** when choosing security patterns

### The Goal

Security should not be an afterthought. By understanding these patterns, you'll:

- ✅ Write secure code from the start
- ✅ Audit code more effectively
- ✅ Understand Solana's unique security model
- ✅ Make informed trade-offs between security and performance
- ✅ Build confidence in your Solana programs

---

## Why Solana Security is Hard

Solana is fundamentally different from Ethereum and other account-based blockchains. Understanding **why** Solana is harder to secure is crucial.

### The Solana Account Model

Unlike Ethereum, where contracts control state:

**Ethereum Model:**
```
Smart Contract → Controls storage → Can authorize state changes
```

**Solana Model:**
```
Program → Receives AccountInfo → Program decides what's authorized
```

In Solana:
- **Programs don't own accounts** — they merely have the *authority* to modify them
- **Anyone can pass any account to your instruction** — it's your job to validate it
- **Accounts are just data containers** with an `owner` field specifying which program controls them
- **Programs must explicitly check every account** — there's no implicit authorization

### The Authorization Gap

This creates a critical difference in security responsibility:

| Aspect | Ethereum | Solana |
|--------|----------|--------|
| **State Owner** | Smart contract | Accounts (separate from program) |
| **Authorization** | Built into contract logic | Must be explicitly verified |
| **Invalid Account** | Impossible (storage is typed) | Possible (any AccountInfo accepted) |
| **Type Confusion** | Prevented by language | Must be prevented by program |
| **Re-initialization** | Prevented by bytecode | Must be prevented by program |

### The Three Security Challenges

#### 1. **Account Ownership is Not Automatic**

In Ethereum:
```solidity
// Storage is controlled by the contract
function transfer(address to, uint256 amount) public {
    balances[msg.sender] -= amount; // msg.sender's data, contract controls it
}
```

In Solana:
```rust
pub fn transfer(ctx: Context<Transfer>, amount: u64) -> Result<()> {
    // ❌ What if `from_account` is a fake token account from another program?
    // The program doesn't automatically own it or control it
    // We must verify the owner ourselves
}
```

#### 2. **Type Safety is Manual**

In Ethereum:
```solidity
// If this contract expects ERC20, it simply won't work with other types
mapping(address => uint256) balances; // Typed storage
```

In Solana:
```rust
// ❌ Without checking, we can't tell if this is really a token account
pub fn accept_token_account(account: AccountInfo) -> Result<()> {
    // Is it owned by the Token program?
    // Does it have the right structure?
    // We don't know until we check!
}
```

#### 3. **State Can Be Re-initialized**

In Ethereum:
```solidity
// Once deployed, bytecode cannot change
// State is always in the expected format
```

In Solana:
```rust
// ❌ Without proper initialization checks, an attacker could:
pub fn initialize(account: &AccountInfo) -> Result<()> {
    // Call this multiple times
    // Resetting state each time
    // Causing fund loss or privilege escalation
}
```

### The Solana Security Model: Design Features

Solana's design is intentional. These challenges exist because:

1. **Flexibility**: Programs can interact in ways Ethereum can't
2. **Composability**: Any program can call any other program
3. **Performance**: Separating state from programs enables parallelization
4. **Transparency**: Nothing is hidden; all data is visible on-chain

These are features, not bugs. But they come with responsibility.

---

## Vulnerability Index

This repository contains **5 concrete examples** of critical vulnerabilities found in real Solana exploits. Each example includes:

- 📝 **Insecure implementation** (demonstrating the vulnerability)
- ✅ **Secure implementation** (showing the fix)
- 📚 **Educational comments** (explaining the vulnerability in depth)
- 🔍 **Attack scenarios** (real ways this could be exploited)

### The 5 Vulnerabilities

| # | Vulnerability | Program | Severity | Real-World Impact | Key Lesson |
|---|---------------|---------|----------|-------------------|-----------|
| **1** | **Missing Signer Check** | `missing-signer` | 🔴 CRITICAL | Unauthorized state changes, privilege escalation | Always verify signatories with `Signer<'info>` |
| **2** | **Missing Owner Check** (Type Cosplay) | `missing-owner-check` | 🔴 CRITICAL | Fake account acceptance, state corruption | Use `Account<'info, T>` for owned accounts |
| **3** | **Arbitrary CPI** | `arbitrary-cpi` | 🔴 CRITICAL | Fund theft, cascading protocol failures | Use `Program<'info, T>` for external programs |
| **4** | **Re-initialization Attack** | `insecure-pda-sharing` | 🔴 CRITICAL | Balance reset, ownership theft, DoS | Use `#[account(init, ...)]` for one-time init |
| **5** | **Integer Overflow** | `integer-overflow` | 🟡 HIGH | Arithmetic errors, silent failures | Use `checked_*` methods for arithmetic |

> **Severity Classification:**  
> 🔴 CRITICAL = Can lead to complete fund loss or protocol compromise  
> 🟡 HIGH = Can lead to data corruption or unexpected state changes

### Quick Links to Examples

Click any example to dive into the code and detailed explanations:

- **[1. Missing Signer Check](./programs/missing-signer/src/lib.rs)** — Demonstrates signature spoofing vulnerability
- **[2. Missing Owner Check](./programs/missing-owner-check/src/lib.rs)** — Demonstrates Type Cosplay attacks
- **[3. Arbitrary CPI](./programs/arbitrary-cpi/src/lib.rs)** — Demonstrates calling malicious programs
- **[4. Re-initialization](./programs/insecure-pda-sharing/src/lib.rs)** — Demonstrates state reset attacks
- **[5. Integer Overflow](./programs/integer-overflow/src/lib.rs)** — [Coming soon - example of unchecked arithmetic]

---

## Repository Structure

```
solana-security-patterns/
│
├── README.md                          # You are here
├── ANCHOR_VS_PINOCCHIO.md            # Framework comparison
│
├── programs/                          # The 5 vulnerability examples
│   ├── missing-signer/
│   │   ├── src/lib.rs                # Signer validation vulnerability
│   │   └── Cargo.toml
│   │
│   ├── missing-owner-check/
│   │   ├── src/lib.rs                # Type Cosplay vulnerability
│   │   └── Cargo.toml
│   │
│   ├── arbitrary-cpi/
│   │   ├── src/lib.rs                # CPI validation vulnerability
│   │   └── Cargo.toml
│   │
│   ├── insecure-pda-sharing/
│   │   ├── src/lib.rs                # Re-initialization vulnerability
│   │   └── Cargo.toml
│   │
│   └── integer-overflow/
│       ├── src/lib.rs                # Arithmetic safety vulnerability
│       └── Cargo.toml
│
├── tests/                            # Integration tests (TBD)
│   └── integration_tests.ts
│
├── Cargo.toml                        # Workspace configuration
└── Anchor.toml                       # Anchor configuration

```

### File Descriptions

| File | Purpose |
|------|---------|
| `README.md` | This file - the entry point for the repository |
| `ANCHOR_VS_PINOCCHIO.md` | Deep dive: Anchor vs low-level Pinocchio comparison |
| `programs/*/src/lib.rs` | The actual vulnerable and secure implementations |
| `Cargo.toml` (root) | Workspace definition (links all 5 programs) |
| `Anchor.toml` | Solana cluster configuration and program IDs |

---

## Quick Start

### Prerequisites

Before you begin, ensure you have:

1. **Rust** (latest stable)
   ```powershell
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
   rustup update
   ```

2. **Solana CLI** (v1.18+)
   ```powershell
   sh -c "$(curl -sSfL https://release.solana.com/v1.18.0/install)"
   ```

3. **Anchor Framework** (v0.29.0+)
   ```powershell
   npm install -g @project-serum/anchor-cli
   ```

4. **Node.js and npm**
   ```powershell
   # Verify installation
   node --version  # v18+
   npm --version   # 9+
   ```

### Verify Installation

```powershell
# Check Rust
rustc --version

# Check Solana
solana --version

# Check Anchor
anchor --version
```

All should show version numbers without errors.

---

## How to Run

### 1. Building the Programs

Build all 5 programs in the workspace:

```powershell
# Navigate to the repository
cd "c:\Users\hp\4k Project\solana-security-patterns"

# Build all programs
anchor build

# Or build individual programs
anchor build --program-name missing-signer
anchor build --program-name missing-owner-check
anchor build --program-name arbitrary-cpi
anchor build --program-name insecure-pda-sharing
anchor build --program-name integer-overflow
```

**What this does:**
- Compiles Rust code to WebAssembly (Solana bytecode)
- Generates IDL (Interface Definition Language) files
- Creates TypeScript client libraries
- Outputs deployable `.so` files in `target/deploy/`

### 2. Running Tests

Run the test suite to verify implementations:

```powershell
# Run all tests
anchor test

# Run tests with verbose output
anchor test --verbose

# Run a specific test file
anchor test tests/missing-signer.ts
```

**Expected output:**
```
✓ Test: Insecure signer check fails when caller is not signer
✓ Test: Secure signer check rejects unauthorized caller
✓ Test: Type cosplay attack fails with Account<'info, T>
...
```

### 3. Testing Individually

Test each program separately:

```powershell
# Build and test missing-signer
cd programs/missing-signer
cargo test --lib

# Build and test missing-owner-check
cd ..\missing-owner-check
cargo test --lib

# Build and test arbitrary-cpi
cd ..\arbitrary-cpi
cargo test --lib
```

### 4. Deploying to Localnet

For more hands-on testing:

```powershell
# Start a local Solana validator
solana-test-validator

# In another terminal, deploy the programs
anchor deploy

# Or deploy individual programs
solana program deploy target/deploy/missing_signer.so
solana program deploy target/deploy/missing_owner_check.so
```

### 5. Code Exploration Workflow

**Recommended learning path:**

1. Start with this README (you are here)
2. Read [ANCHOR_VS_PINOCCHIO.md](./ANCHOR_VS_PINOCCHIO.md) for framework context
3. Examine `programs/missing-signer/src/lib.rs`
   - Read insecure implementation first
   - Then read secure implementation
   - Read all comments carefully
4. Move to the next vulnerability in order
5. For each: compare insecure vs secure, understand the attack scenario
6. Build locally to verify code compiles
7. Write your own test cases

---

## Key Takeaways

### 1. Solana's Account Model Requires Explicit Verification

**The Core Insight:**

In Solana, security is a *feature of the program*, not a feature of the runtime. Unlike Ethereum where the runtime prevents invalid state transitions, Solana programs must implement all security checks themselves.

```rust
// ✅ Always verify:
// - Who is authorized (Signer<'info>)?
// - Does the account belong to us (Account<'info, T>)?
// - Is the external program trusted (Program<'info, T>)?
// - Is this the first initialization (#[account(init, ...)])?
```

### 2. Anchor Eliminates Most Categories of Vulnerabilities

**What Anchor Does:**

Anchor is not just a convenience library—it's a *security framework*. By using Anchor's type system:

- `Account<'info, T>` — Prevents Type Cosplay attacks
- `Signer<'info>` — Enforces signature verification
- `Program<'info, T>` — Ensures correct CPI targets
- `#[account(init, ...)]` — Prevents re-initialization
- Discriminators — Prevent type confusion

**Reality Check:**

Anchor doesn't prevent all bugs. It prevents *categories* of bugs:

```rust
// ✅ Anchor prevents this class of vulnerabilities:
// - Calling the wrong program
// - Processing fake accounts
// - Re-initializing existing accounts

// ⚠️ Anchor doesn't prevent:
// - Logic bugs in your instructions
// - Integer overflows in custom arithmetic
// - Incorrect permission checks in constraints
// - Business logic errors (off-by-one in calculations)
```

### 3. Abstractions Help, But Aren't Magic

**The Abstraction Trade-off:**

Anchor provides safety guarantees, but at a cost:

```
┌─────────────────────────────────────────────────┐
│ Security ◄────────────────────────────► Performance
│                                                 │
│ ✅ Anchor: High security, moderate performance │
│ ⚡ Pinocchio: High performance, manual security│
│ 🎯 Sweet spot: Use Anchor, optimize hot paths │
└─────────────────────────────────────────────────┘
```

**When to use each:**

| Use Anchor for | Use Low-level for |
|----------------|-------------------|
| Account validation | Inner loops |
| Permission checks | Data serialization |
| Type safety | Hot paths |
| Most instructions | Specialized optimization |

### 4. Security is a Practice, Not Just Code

**Building Secure Programs:**

Writing secure Solana code requires:

1. **Knowledge**: Understand the 5 vulnerability categories
2. **Pattern Recognition**: Know the secure patterns (use `Account<T>`, `Signer<T>`, etc.)
3. **Code Review**: Have others review your security-critical code
4. **Testing**: Write tests for edge cases and attack scenarios
5. **Auditing**: Get professional security audits for production code

**The Mindset:**

```rust
// ❌ Don't think like this:
// "I'll build it, then make it secure"

// ✅ Think like this:
// "I'm building it securely from the start"

// Write secure code:
#[derive(Accounts)]
pub struct SecureInstruction {
    #[account(mut, constraint = is_authorized(&ctx))]
    pub state: Account<'info, State>,
    pub authority: Signer<'info>,
}

// Then test the authorization:
#[test]
fn test_unauthorized_call_fails() {
    // Attack attempt should fail
}
```

### 5. The Anchor Checklist for Every Instruction

Before you ship any Solana program, ask:

- ✅ Does each account have the correct type (`Account<T>` vs `AccountInfo`)?
- ✅ Are signers verified (`Signer<'info>`)?
- ✅ Are external programs verified (`Program<'info, T>`)?
- ✅ Is initialization protected (`#[account(init, ...)]`)?
- ✅ Are permissions checked (`constraint = ...`)?
- ✅ Are arithmetic operations safe (`checked_add`, etc.)?
- ✅ Have I documented `/// CHECK:` for any `AccountInfo`?
- ✅ Would my constraints prevent an attacker from calling this?

If any answer is "no", reconsider your design.

---

## Learning Path

### For Beginners: Start Here

1. **Read this README** (especially "Why Solana Security is Hard")
2. **Study one example at a time:**
   - Read the insecure version
   - Understand the vulnerability
   - Read the secure version
   - Understand the fix
3. **Build locally** to verify everything works
4. **Write a test case** for each vulnerability

**Estimated time**: 2-3 hours

### For Experienced Developers: Speed Run

1. Skim "Why Solana Security is Hard"
2. Read [ANCHOR_VS_PINOCCHIO.md](./ANCHOR_VS_PINOCCHIO.md)
3. Scan the 5 examples, focusing on comments
4. Run `anchor build && anchor test`
5. Reference as needed for code reviews

**Estimated time**: 30-45 minutes

### For Auditors: Deep Dive

1. Read all of this README carefully
2. Study each program's lib.rs in depth
3. Read ANCHOR_VS_PINOCCHIO.md
4. Write attack tests for each vulnerability
5. Create a threat model for your own protocols

**Estimated time**: 4-6 hours

### For Teams: Group Learning

1. **Day 1**: Introduction and account model (1 hour)
2. **Day 2-4**: One vulnerability example per day (1.5 hours each)
3. **Day 5**: Team security review of internal codebase (2 hours)
4. **Ongoing**: Reference materials during development

---

## Resources

### Official Documentation

- **[Anchor Book](https://book.anchor-lang.com/)** — Official Anchor documentation
- **[Solana Docs](https://docs.solana.com/)** — Official Solana documentation
- **[Solana Program Library (SPL)](https://github.com/solana-labs/solana-program-library)** — Reference implementations

### Security Resources

- **[Solana Security Best Practices](https://developers.solana.com/docs/security)** — Official security guide
- **[Runtime Warnings](https://developers.solana.com/docs/programs/debugging#runtime-warnings)** — Understanding program errors
- **[Anchor Security Advisories](https://github.com/coral-xyz/anchor/security/advisories)** — Known Anchor issues

### Community

- **[Solana Stack Exchange](https://solana.stackexchange.com/)** — Q&A community
- **[Solana Discord](https://discord.gg/solana)** — Official Discord
- **[Anchor Discord](https://discord.gg/anchor)** — Anchor framework community

### Recommended Reading

- **"The Solana Programming Model"** by Solana Labs
- **"How to Audit a Solana Program"** by Halborn
- **"Common Solana Vulnerabilities and How to Prevent Them"** by Trail of Bits

### Tools

- **[Anchor CLI](https://github.com/coral-xyz/anchor)** — Framework CLI
- **[Solana CLI](https://github.com/solana-labs/solana)** — Blockchain CLI
- **[Solend Audits](https://github.com/solendprotocol/audits)** — Examples of audited code

---

## Contributing

This repository is intended as an educational resource. If you:

- Find an error or unclear explanation
- Have suggestions for additional examples
- Want to contribute test cases
- Identify a vulnerability we missed

Please feel free to contribute! This is a living document meant to improve over time.

---

## Disclaimer

This repository is **educational only**. The code examples demonstrate vulnerabilities and fixes for learning purposes. While the secure patterns are best practices:

1. **Always get professional audits** before deploying to mainnet
2. **Never copy code directly** without understanding it
3. **Test thoroughly** in localnet and testnet first
4. **Security is a process**, not just code patterns

The examples here are simplified for clarity. Real programs may need additional checks.

---

## About This Repository

**Created for**: Superteam Bounty - Solana Security Reference Implementation

**Purpose**: To provide a comprehensive, code-based reference for understanding common Solana vulnerabilities and how Anchor prevents them.

**Philosophy**: Security should be the default, not an afterthought. By making secure patterns the obvious choice, we build a safer Solana ecosystem.

---

## Summary

Solana's account model is powerful but requires developers to think differently about security. This repository provides:

1. **Understanding** of why Solana is harder than other blockchains
2. **Examples** of 5 critical vulnerabilities
3. **Solutions** using Anchor best practices
4. **Education** on when and why to use different patterns
5. **Resources** for continued learning

**Your next step**: Pick one example and dive in. The code is well-commented, and the patterns are meant to be learned, understood, and internalized.

---

**Happy secure coding!** 🚀

*Built with ❤️ for the Solana developer community*

---

## Quick Reference

### The 5 Vulnerabilities at a Glance

| # | Name | Pattern | Fix |
|---|------|---------|-----|
| 1 | Missing Signer | Using `AccountInfo` for authorization | Use `Signer<'info>` |
| 2 | Type Cosplay | Accepting `AccountInfo` without type check | Use `Account<'info, T>` |
| 3 | Arbitrary CPI | Not validating called program | Use `Program<'info, T>` |
| 4 | Re-initialization | No check if already initialized | Use `#[account(init, ...)]` |
| 5 | Integer Overflow | Unchecked arithmetic | Use `checked_*` methods |

### Essential Anchor Types

```rust
// For signers
pub authority: Signer<'info>

// For owned accounts
pub account: Account<'info, MyStruct>

// For external programs
pub token_program: Program<'info, Token>

// For one-time initialization
#[account(init, payer = payer, space = 8 + 32)]
pub new_account: Account<'info, MyStruct>

// For custom checks
#[account(constraint = is_valid(&self))]
pub validated: Account<'info, MyStruct>
```

### Build Commands

```powershell
# Development
anchor build
anchor test
anchor deploy

# Production
anchor build --release
anchor deploy --provider.cluster mainnet
```

---

*Last Updated: January 2026*  
*Anchor Version: 0.29.0*  
*Solana Version: 1.18+*
