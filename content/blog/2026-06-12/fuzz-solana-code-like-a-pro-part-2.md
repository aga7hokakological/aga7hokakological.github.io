+++
title = "Fuzz solana programs like a Pro - Part 2"
date = 2026-06-12
draft = false
tags = ["solana", "fuzzer", "anchor", "invariants"]
+++


### Introduction

In the part 1 we set up Trident and got basic fuzzing running.

You can check it [here.](https://aga7hokakological.github.io/blog/2026-02-18/fuzz-solana-code-like-a-pro-part-1/)

Single instructions, random inputs, then watching it crash. That gets you far but real DeFi bugs don't live in single instructions. 
Today we'll be moving one step forward and learn how to write invariants and multi-instruction transactions. 

We'll use a real lending platform as the target.
The code used for this blog is [here.](https://github.com/aga7hokakological/trident-examples/blob/main/lending-platform/trident-tests/fuzz_0/test_fuzz.rs)

As you can see the code has 4 basic functions (`deposit`, `borrow`, `withdraw`, `repay`) along with initialization of the program like how any general
lending platform has them.
Just like previous blog we start with understanding how accounts are present. As you can see we have 4 accounts and system program.
```rust
#[derive(Default)]
pub struct AccountAddresses {
    pub pool: AddressStorage,

    pub position: AddressStorage,

    pub user: AddressStorage,

    pub system_program: AddressStorage,

    pub authority: AddressStorage,
}
```

Now lets move to `test_fuzz.rs` file. Like general flow where `start()` function is the first one to execute we need to declare the accounts and initialize 
program.

```rust
let admin = self.fuzz_accounts.authority.insert(&mut self.trident, None);
self.trident.airdrop(&admin, 1500 * LAMPORTS_PER_SOL);

// Intialize the pool
let pool = self.fuzz_accounts.pool.insert(
    &mut self.trident,
    Some(PdaSeeds::new(
        &[
            b"lending_pool",
        ],
        pubkey!("YOUR_PROGRAM_ID")
    ))
);
```

Similarly by checking which accounts need `pda` derivation we need to initialize them too. And then process the transactions.

### Multi-Instruction transactions

Flows are where the real fuzzing happens.
Each `#[flow]` is an independent function the fuzzer can call zero or more times, in any order.

```rust
#[flow_executor]
impl FuzzTest {
    // ...

    #[flow]
    fn flow1(&mut self) { 
        // Declare your functionality
    }   // randomly selected and repeated

    #[flow]
    fn flow2(&mut self) {
        // Declare your functionality
    }   // randomly selected and repeated

    #[flow]
    fn flow3(&mut self) { 
        // Declare your functionality
    }   // randomly selected and repeated

    // ...
}
```

The fuzzer might run: `flow2 → flow1 → flow2 → flow2 → flow2` where say., `flow1` is `deposit` and `flow2` is `borrow` function call. 
Each call is a separate transaction. State accumulates. The user's `position.borrowed` 
grows across multiple borrow calls, never reset between flows until the iteration ends and state is wiped.

This is how the double-borrow bug gets triggered: the fuzzer finds a sequence where two borrow calls together exceed what a single check allows.

We have `4` different functions present in our solana program and sometimes you might want to see how program behaves with multiple transaction in single flow. 
Trident has solution for that too.

Trident supports executing multiple instructions within a single transaction using the `process_transaction` method with an array of instructions.
```rust
    #[flow]
    fn flow3(&mut self) {
        let amount = self.trident.random_from_range(1..25*LAMPORTS_PER_SOL);
        let borrow_amt = self.trident.random_from_range(1..25*LAMPORTS_PER_SOL);
        let user = self.fuzz_accounts.user.get(&mut self.trident).unwrap();

        let position = self.fuzz_accounts.position.get(&mut self.trident).unwrap();

        let pool = match self.fuzz_accounts.pool.get(&mut self.trident) {
            Some(v) => v,
            None => return,
        };

        // First transaction
        let depositTrxs = DepositInstruction::data(DepositInstructionData::new(amount))
            .accounts(DepositInstructionAccounts::new(pool, position, user))
            .instruction();

        // Second transaction
        let borrowTrxs = BorrowInstruction::data(BorrowInstructionData::new(borrow_amt))
            .accounts(BorrowInstructionAccounts::new(pool, position, user))
            .instruction();

        let res = self.trident.process_transaction(&[depositTrxs, borrowTrxs], Some("Deposit->Borrow"));
    }
```

As you can see above in the code, calling `process_transaction` allows you to merge different transactions together:
```rust
let res = self.trident.process_transaction(&[depositTrxs, borrowTrxs], Some("Deposit->Borrow"));
```

The two instructions (`deposit`, `borrow`) run atomically: if either fails, both roll back. This matters here because you cannot borrow before deposit.
You need some collateral to be able to borrow amount from the protocol.

**Why do we need them:**
- Atomic Operations: All instructions succeed or fail together
- Complex Workflows: Test instruction sequences that depend on each other
- State Consistency: Ensure related operations maintain program invariants

#### get vs insert in Flows
```RUST
// In start(): use insert — creates/registers the account
let pool = self.fuzz_accounts.pool.insert(&mut self.trident, Some(PdaSeeds::new(...)));

// In flows: get retrieves the already-registered address
let pool = self.fuzz_accounts.pool.get(&mut self.trident).unwrap();

// Position uses insert even in flows — idempotent PDA derivation
let position = self.fuzz_accounts.user_position.insert(&mut self.trident, Some(PdaSeeds::new(...)));
```

`insert` with `PdaSeeds` is safe to call multiple times for the same PDA. It derives the same address deterministically. `get` is used for accounts where you just want back the pubkey you already stored. Use `get` for singleton accounts (`pool`, `authority`). Use `insert` for PDAs you re-derive each flow.

### Invariants

Now let's move on to understanding most critical part that is writing invariants.
The code used for this blog has some bugs already injected in it so that we can do invariant testing.

Multi-instruction flows let the fuzzer reach buggy states. On the other hand invariants let it recognize them.

Without an invariant, the fuzzer only catches panics and program errors. Our collateral bug doesn't panic then 
the program happily lets you borrow 100% of collateral and returns `Ok(())`. To catch it, we must define what "correct" looks like and assert it.

The invariant that we are trying to break here is `position borrowed should always be less than deposited amount`

Now if we tried to run the code without adding invariant we might not get anything. There will be few errors but they don't show
any bugs. 

Here is how you can write invariant in end flow:
```rust
const EXPECTED_COLLATERAL_FACTOR_BPS: u64 = 8_000;
const BPS_DIVISOR: u64 = 10_000;

let Some(position_pubkey) = self.fuzz_accounts.position.get(&mut self.trident) else {
    return;
};

let Some(position) = self.trident.get_account_with_type::<UserPosition>(&position_pubkey, 8) else {
    return;
};

let max_borrow = position
    .deposited
    .saturating_mul(EXPECTED_COLLATERAL_FACTOR_BPS)
    / BPS_DIVISOR;

assert!(
    position.borrowed <= max_borrow,
    "INVARIANT VIOLATED: borrowed {} > max_borrow {} (deposited={}, collateral={}%)",
    position.borrowed,
    max_borrow,
    position.deposited,
    EXPECTED_COLLATERAL_FACTOR_BPS / 100,
);
```

After that if you run the program you will get output like following:

![alt](/blog/2026-06-12/fuzzp.png)

When `assert!` panics, Trident catches it, records the input sequence that triggered it, and writes a crash file. You get the exact call sequence that violated the invariant.

#### Reading On-Chain State: get_account_with_type

```rust
let Some(position) = self.trident.get_account_with_type::<UserPosition>(&position_pubkey, 8) else {
    return;
};
```
The `8` is the Anchor discriminator offset. Anchor prepends `8` bytes to every account for type identification. 
`get_account_with_type` skips those bytes and deserializes the rest as `UserPosition` via Borsh. The type must implement `BorshDeserialize`, which Anchor's `#[account]` macro derives automatically.

#### What the Invariant Catches

With `COLLATERAL_FACTOR_BPS = 10_000` in the program and `EXPECTED_COLLATERAL_FACTOR_BPS = 8_000` in our invariant, the fuzzer will find inputs where:
```
position.borrowed > (position.deposited * 8_000) / 10_000
```

This happens on the very first successful borrow if the user deposits `X` and borrows more than `0.8 * X`. The program allows it (100% factor). The invariant rejects it (80% expected) and then it crashes.

The double-borrow bug makes this even more accessible: deposit X, borrow X, borrow X again. Now `borrowed = 2X`, `deposited = X`. The invariant fires immediately.

#### Other way of writing invariants 

There is one more question remains? Do invariants need to be written in `#[end]` section?
Well not necessarily. You can use the **"before/after"** pattern to write invariants.

```rust
#[flow]
fn flow2(&mut self) {
    let amount = self.trident.random_from_range(1..25 * LAMPORTS_PER_SOL);
    let user = self.fuzz_accounts.user.get(&mut self.trident).unwrap();
    let pool = self.fuzz_accounts.pool.get(&mut self.trident).unwrap();

    let position_pubkey = self.fuzz_accounts.user_position.insert(
        &mut self.trident,
        Some(PdaSeeds::new(
            &[b"user_position", user.as_ref()],
            pubkey!("YOUR_PROGRAM_ID"),
        )),
    );

    // Capture state BEFORE borrow
    let before = self
        .trident
        .get_account_with_type::<UserPosition>(&position_pubkey, 8);

    let ix = BorrowInstruction::data(BorrowInstructionData::new(amount))
        .accounts(BorrowInstructionAccounts::new(pool, position_pubkey, user))
        .instruction();

    let res = self.trident.process_transaction(&[ix], Some("Borrow"));

    // Only check if tx succeeded AND we had pre-state
    if res.is_success() {
        if let Some(before) = before {
            let after = self
                .trident
                .get_account_with_type::<UserPosition>(&position_pubkey, 8)
                .expect("position must exist after successful borrow");

            const EXPECTED_COLLATERAL_FACTOR_BPS: u64 = 8_000;
            const BPS_DIVISOR: u64 = 10_000;

            let max_borrow = after
                .deposited
                .saturating_mul(EXPECTED_COLLATERAL_FACTOR_BPS)
                / BPS_DIVISOR;

            assert!(
                after.borrowed <= max_borrow,
                "INVARIANT VIOLATED after Borrow: borrowed={} > max={} (deposited={}, delta_borrow={})",
                after.borrowed,
                max_borrow,
                after.deposited,
                after.borrowed.saturating_sub(before.borrowed),
            );
        }
    }
}
```

Then is there an advantage using one over another? Nope.
Here is comparison:

|                           | `#[end]` block   | `before/after` flow |
| --------                  | --------         | --------            |
| Granularity               | End of iteration | per-transaction     |
| pinpointing failed txs    | No               | Yes                 |
| catching failed state     | Yes              | Yes                 |
| Error message             | Final state only | Shows data too      |

This table shows that it is better to use both rather than single way to write invariants. `#[end]` will catch anything flows miss; per-flow will catch exact tx.
It'll lead to better coverage.

#### Writing Good Invariants
A few principles:

**Assert the business rule, not the code**: Don't read the constant from the program — encode what the constant should be. The whole point is to catch when the implementation diverges from the specification.

**Handle missing accounts gracefully**: If the position doesn't exist, the invariant trivially holds — early return, not panic. A missing account is not a bug.

**Check global pool state too**: A per-user invariant is good. A global invariant is better:

```rust
// Global invariant: pool is solvent
let pool = self.trident.get_account_with_type::<LendingPool>(&pool_pubkey, 8).unwrap();
assert!(
    pool.total_borrows <= pool.total_deposits,
    "Pool insolvent: borrows {} > deposits {}",
    pool.total_borrows,
    pool.total_deposits
);
```

**Invariants fire at end, not mid-flow**: Intermediate states can be temporarily invalid (e.g., a multi-step atomic operation). Only assert what must be true after every complete operation sequence.

#### Putting It All Together

Here's the complete lifecycle of one fuzz iteration:
```
start()
  ├── airdrop admin + user
  ├── derive PDAs (pool, positions)
  ├── initialize pool [1 tx]
  └── deposit admin + deposit user [1 atomic tx]

fuzzer randomly picks N flows:
  ├── flow1() → deposit random amount [1 tx]
  ├── flow2() → borrow random amount [1 tx]
  ├── flow2() → borrow random amount [1 tx]  ← accumulates borrowed
  └── ... (N more)

end()
  ├── read UserPosition from chain
  └── assert borrowed ≤ deposited * 80%  ← PANIC if violated → crash file written

[account state wiped]
[next iteration begins]
```

Run with:
```
trident fuzz run fuzz_0
```
The fuzzer will report a crash within seconds for this particular bug. The state space is small enough that random sequences hit it quickly.

### Reading the Crash Output
When the invariant fires, you'll see something like:
```
INVARIANT VIOLATED: borrowed 5000000000 > max_borrow 4000000000 (deposited=5000000000, collateral=80%)
```

Trident writes the reproducing input to `trident-tests/fuzz_0/artifacts/`. Replay it:
```
trident fuzz run-file fuzz_0 ./artifacts/crash-<hash>
```
The replay re-runs the exact sequence of flows and amounts that triggered the bug.

### Conclusion

This might be an intentional bug we found in the code. But in real protocols such a bugs might survive easily if we just did manual review and testing.
Invariants are the difference between a fuzzer that finds crashes and a fuzzer that finds bugs. Without them, Trident only catches what the runtime catches such panics, arithmetic traps, explicit program errors. But most DeFi vulnerabilities don't crash. The program returns `Ok(())`, the transaction succeeds, the attacker walks away with funds. An invariant is how you encode the rule the program should enforce, independent of whether the program actually does. Write them from the specification, not the code. If you derive the constant from the program, you'll miss the case where the constant itself is wrong. Keep them simple: a well-chosen assertion like `borrowed <= deposited * 0.8` is worth more than a thousand random inputs without one. The fuzzer's job is to explore state space. Your job is to define what `"wrong"` looks like. Do both, and automated fuzzing becomes a genuine security primitive and not just a way to find the bugs you already suspected, but the ones you never thought to check. 
