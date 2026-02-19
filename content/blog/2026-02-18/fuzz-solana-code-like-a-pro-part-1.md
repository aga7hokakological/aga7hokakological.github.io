+++
title = "Fuzz solana programs like a Pro - Part 1"
date = 2026-02-18
draft = false
tags = ["solana", "fuzzer", "anchor"]
+++

Smart contract exploits rarely come from obvious code paths. They emerge from unexpected edge case sequences of state transitions that traditional tests never explore.
Most of the smart contracts today rely heavily on unit tests. Unit tests validate correctness along expected flows. Attackers don’t follow expected flows. This is where fuzzing becomes indispensable.

In this article, we’ll walk through how to write real, invariant-driven fuzz tests using Trident, a purpose-built fuzzing framework for Solana programs.

### Why Fuzzing Matters in Solana Security ?

Unit tests validate known scenarios but miss edge cases, while stateless fuzzing adds randomness without modeling real protocol logic.
Stateful fuzzing simulates sequences of actions and state transitions, uncovering realistic economic and logic bugs.
DeFi protocols are like state machines. Bugs do not emerge from single calls, but from sequences of complex operations. That’s why stateful fuzzing is essential.

### What is Trident?
Trident is a Rust-native fuzzing framework built specifically for Solana programs and Anchor-based protocols. It was developed by [Ackee Blockchain Security](https://ackee.xyz/) to bring invariant-driven fuzzing to the Solana ecosystem.

#### Unlike generic fuzzers, Trident understands:
* Solana accounts
* Program instructions
* Multi-user state transitions
* Deterministic replay

It allows you to write structured fuzz tests that simulate adversarial user behavior while validating protocol invariants.

#### Mental Model of a Trident Fuzz Test
Before writing code, build the right mental model.
A Trident fuzz test has four core components:
* Fuzz target — orchestrates execution
* Handlers — simulate user actions
* Invariants — safety properties that must hold
* Multi-Instruction transaction — Execute multiple instructions in a single transaction

Conceptually:

> Random Input → Action → Program Call → State Change → Invariant Check

The fuzzer generates inputs, handlers translate them into meaningful actions, and invariants validate correctness. This separation of responsibilities is key to writing effective fuzz tests.

#### Setup and Installation
You’ll need a typical Solana development environment:
[Rust toolchain](https://rust-lang.org/tools/install/), 
[Solana CLI](https://solana.com/docs/intro/installation), 
[Anchor framework](https://solana.com/docs/intro/installation/anchor-cli-basics)

Install Trident:
```rust
cargo install trident-cli
```

Verify installation:
```rust
trident --version
```

#### Introducing the Vault Example
To make this practical, let’s take a example of simple vault program.

```rust
use anchor_lang::prelude::*;
use anchor_lang::system_program::{transfer, Transfer};

declare_id!("Your program ID");

#[program]
pub mod vault {
    use super::*;

    pub fn deposit(ctx: Context<VaultAction>, amount: u64) -> Result<()> {
        // deposit logic
        // Check if vault is empty
        require_eq!(ctx.accounts.vault.lamports(), 0, VaultError::VaultAlreadyExists);

        // Ensure amount exceeds rent-exempt minimum
        require_gt!(amount, Rent::get()?.minimum_balance(0), VaultError::InvalidAmount);

        transfer(
            CpiContext::new(
                ctx.accounts.system_program.to_account_info(),
                Transfer {
                    from: ctx.accounts.signer.to_account_info(),
                    to: ctx.accounts.vault.to_account_info(),
                },
            ),
            amount,
        )?;
        Ok(())
    }

    pub fn withdraw(ctx: Context<VaultAction>) -> Result<()> {
        // withdraw logic
        // Check if vault has any lamports
        require_neq!(ctx.accounts.vault.lamports(), 0, VaultError::InvalidAmount);

        // Create PDA signer seeds
        let signer_key = ctx.accounts.signer.key();
        let signer_seeds = &[b"vault", signer_key.as_ref(), &[ctx.bumps.vault]];
        // Transfer all lamports from vault to signer
        transfer(
            CpiContext::new_with_signer(
                ctx.accounts.system_program.to_account_info(),
                Transfer {
                    from: ctx.accounts.vault.to_account_info(),
                    to: ctx.accounts.signer.to_account_info(),
                },
                &[&signer_seeds[..]]
            ),
            ctx.accounts.vault.lamports()
        )?;
        
        Ok(())
    }
}

#[derive(Accounts)]
pub struct VaultAction<'info> {
    #[account(mut)]
    pub signer: Signer<'info>,
    #[account(
        mut,
        seeds = [b"vault", signer.key().as_ref()],
        bump,
    )]
    pub vault: SystemAccount<'info>,
    pub system_program: Program<'info, System>,
}

#[error_code]
pub enum VaultError {
    #[msg("Vault already exists")]
    VaultAlreadyExists,
    #[msg("Invalid amount")]
    InvalidAmount,
}
```

The vault supports:
* Token deposits
* Token withdrawals
* Internal accounting

Even simple vaults can hide subtle accounting bugs when subjected to adversarial sequences. Currently I am not using vulnerable vault code just want to show you flow of fuzzing.

#### Designing the Fuzz Test Setup 
In the root of your vault program repo run the following trident command:
```rust 
trident init 
```

If the command passes then it'll create `trident-tests` folder with following structure:
```
project-root
├── trident-tests
│   ├── .fuzz-artifacts         # Fuzzing artifacts (dashboard, metrics, etc.)
│   ├── fuzz_0                  # Your first fuzz test
│   │   ├── test_fuzz.rs        # Main fuzz test logic
│   │   ├── fuzz_accounts.rs    # Account addresses storage
│   │   └── types.rs            # IDL-like generated types
│   ├── fuzz_1                  # Additional fuzz tests
│   ├── fuzz_X                  # Multiple fuzz tests supported
│   ├── fuzzing                 # Compilation and crash artifacts
│   ├── Cargo.toml              # Rust dependencies
│   └── Trident.toml            # Trident configuration
└── ...
```

The `fuzz_accounts.rs` file contains all the accounts that are in the program code. The `types.rs` file contains IDL-like generated types of the complete program.

Currently we only care about `test_fuzz.rs` file present in the `fuzz_0` folder. The current empty `test_fuzz.rs` might look something like this:
```rust
    #[init]
    fn start(&mut self) {
        // perform any initialization here, this method will be executed
        // at start of each iteration
    }

    #[flow]
    fn flow1(&mut self) {
        // perform logic which is meant to be fuzzed
        // this flow is selected randomly from other flows
    }

    #[end]
    fn end(&mut self) {
        // perform any cleaning here, this method will be executed
        // at the end of each iteration
    }
```
As stated in comments of each attribute `#[init]` with start function is to intialize required minimal setup with which we can start fuzzing. 
With `#[flow]` you can decide which function to performed fuzz on. You can have multiple flows. 
The `#[end]` means any ending function like closing account etc can be executed here.

#### Starting with writing fuzz test
Add following code in the `start()` function of `#[init]` block:

```rust
    #[init]
    fn start(&mut self) {
        let signer = self.fuzz_accounts.signer.insert(&mut self.trident, None);
        self.trident.airdrop(&signer, 150 * LAMPORTS_PER_SOL);
    }
``` 

As I said above we only need minimal setup to start actual fuzzing. In the block we are initializing the `signer` address from `fuzz_account.rs` file. Also airdroping some lamports for rent/transfer purposes.

Now add following code in the `flow1()` function of `#[flow]` block:
```rust
    #[flow]
    fn flow1(&mut self) {
        let signer = self.fuzz_accounts.signer.get(&mut self.trident).unwrap();

        let vault = self.fuzz_accounts.vault.insert(
            &mut self.trident,
            Some(PdaSeeds::new(
                &[
                    b"vault",
                    signer.as_ref(),
                ],
                pubkey!("Your program ID"),
            ))
        );

        let account = self.trident.get_account(&vault);
        let vault_account_size = account.data().len();
        let rent_exempt_min = Rent::default().minimum_balance(vault_account_size);
        let amount = self.trident.random_from_range(rent_exempt_min + 1..100*LAMPORTS_PER_SOL);

        let ix = DepositInstruction::data(DepositInstructionData::new(amount))
            .accounts(DepositInstructionAccounts::new(signer, vault))
            .instruction();

        let res = self.trident.process_transaction(&[ix], Some("Deposit"));
    }
```
In the first line we are getting `signer` from the `start()` function. On the next line we are initializing `vault` account with its seeds. If you check our program again then you can see that we need rent to be provided to the `vault` accounts that we are going to be initializing so that's what next four lines of code are for.

In the next line the instruction is created. To create instruction you need to check `types.rs` file. There you can see implementation of each function instruction. In `vault` program in `deposit()` function we are taking amount as parameter so that why we populate `DepositInstructionData::new()` with `amount`. Also we need to provide two accounts (`signer`, `vault`) to `DepositInstructionAccounts::new()`.

Now we are remain with our last function which is `withdraw()`. Let's change `flow2()` function like this:
```rust
    #[flow]
    fn flow2(&mut self) {
        let signer = self.fuzz_accounts.signer.get(&mut self.trident).unwrap();
        let vault = self.fuzz_accounts.vault.get(&mut self.trident).unwrap();

        let ix = WithdrawInstruction::data(WithdrawInstructionData::new())
            .accounts(WithdrawInstructionAccounts::new(signer, vault))
            .instruction();

        let res = self.trident.process_transaction(&[ix], Some("Withdraw"));
    }
```

Similar to `flow1()` function as we have taken signer from `start()` same we do here. Our simple fuzzing flow and setup is done.

Now to test fuzzing run the following command from the `trident-tests` folder:
```rust
trident fuzz run fuzz_0
```

After running the command you might see something like this:
![alt](/blog/2026-02-18/vault-first-test-run.png)

In the bottom table you see instructions that we processed, total instructions that were invoked in fuzzing, success and failed rate of instructions and lastly panicked instructions.

But before that you're seeing some failed assertion in `flow2()` function. Let's fix that by changing the `getter` of the `fuzz_accounts` for `vault` like this:
```rust
    let vault = self.fuzz_accounts.vault.get(&mut self.trident) {
        Some(v) => v,
        None => return,
    };
``` 

The particular assertion fails because in some runs `flow2()` can be selected before `flow1()`. As `insert` is called in `flow1()`, if `flow2()` is triggered first then it results in assertion failure because `vault` is initialize only in `flow1()`.

Now if we run the tests we won't get the error. I think we are done here. This is it mostly for the program though we can see that the failed rate is almost 50% percent. Let's log the data and debug what is causing it.
For that run following command:
```rust
TRIDENT_LOG=1 trident fuzz run fuzz_0
``` 

After running the command you might see something like this:
![alt](/blog/2026-02-18/vault-debug.png)

As we can see there are still some issues because of which the instruction failure rate is high. Let's see what is wrong:
```log
[2026-02-19T11:06:13.111022437Z DEBUG solana_runtime::message_processor::stable_log] Program 7hpQ5PGwMzKqGTrbPJYP1B2QMSHoxUgF68aZwWVDTJXH consumed 9716 of 1400000 compute units
[2026-02-19T11:06:13.111025643Z DEBUG solana_runtime::message_processor::stable_log] Program 7hpQ5PGwMzKqGTrbPJYP1B2QMSHoxUgF68aZwWVDTJXH failed: custom program error: 0x1771
[2026-02-19T11:06:13.111058635Z DEBUG solana_runtime::message_processor::stable_log] Program 7hpQ5PGwMzKqGTrbPJYP1B2QMSHoxUgF68aZwWVDTJXH invoke [1]
[2026-02-19T11:06:13.111066390Z DEBUG solana_runtime::message_processor::stable_log] Program log: Instruction: Deposit
[2026-02-19T11:06:13.111092459Z DEBUG solana_runtime::message_processor::stable_log] Program 11111111111111111111111111111111 invoke [2]
[2026-02-19T11:06:13.111095965Z DEBUG solana_runtime::message_processor::stable_log] Program 11111111111111111111111111111111 success
[2026-02-19T11:06:13.111107587Z DEBUG solana_runtime::message_processor::stable_log] Program 7hpQ5PGwMzKqGTrbPJYP1B2QMSHoxUgF68aZwWVDTJXH consumed 9601 of 1400000 compute units
[2026-02-19T11:06:13.111110963Z DEBUG solana_runtime::message_processor::stable_log] Program 7hpQ5PGwMzKqGTrbPJYP1B2QMSHoxUgF68aZwWVDTJXH success
[2026-02-19T11:06:13.111144636Z DEBUG solana_runtime::message_processor::stable_log] Program 7hpQ5PGwMzKqGTrbPJYP1B2QMSHoxUgF68aZwWVDTJXH invoke [1]
[2026-02-19T11:06:13.111152171Z DEBUG solana_runtime::message_processor::stable_log] Program log: Instruction: Deposit
[2026-02-19T11:06:13.111178520Z DEBUG solana_runtime::message_processor::stable_log] Program log: AnchorError thrown in programs/vault/src/lib.rs:13. Error Code: VaultAlreadyExists. Error Number: 6000. Error Message: Vault already exists.
```

The error says `vault` already exists: 
```log
Program log: AnchorError thrown in programs/vault/src/lib.rs:13. Error Code: VaultAlreadyExists. Error Number: 6000. Error Message: Vault already exists.
```

The `vault` program checks `require_eq!(vault.lamports(), 0, VaultAlreadyExists)`. So when `flow1()` runs again without `withdraw` in between, the `vault` still holds the lamports from the previous `deposit()`.

To fix this add following lines after `get_account(&vault)`:
```rust
    // Skip if vault already funded
    if account.lamports() > 0 {
        return;
    }
```

Now that this is done run the fuzz command again and you'll see that there are no failures while `deposit()`. Now we still can see that failure rate for `withdraw()` is still there. The failure log says:
```log
Program log: AnchorError thrown in programs/vault/src/lib.rs:34. Error Code: InvalidAmount. Error Number: 6001. Error Message: Invalid amount.
```

Now the error says invalid amount that means the program is fuzzing `withdraw()` function with wrong amount. If we check the `withdraw()` function again we can see that there is require statement `require_neq!(ctx.accounts.vault.lamports(), 0, VaultError::InvalidAmount);`. It states that if there are no lamports then transaction should fail. 

Accordingly if `flow2()` is selected first before `flow1()` then because there are no lamports in `vault` the instruction fails. 
We already have fixed error in deposit can you do it for `withdraw()`?

After fixing if we run fuzzer then we won't be seeing any errors. 
![alt](/blog/2026-02-18/vault-success-run.png)

You can check the complete repo [here](https://github.com/aga7hokakological/trident-examples)

In the next post we will venture into more things like finding bugs, invariants, etc(coming soon)..

### Final Thoughts

Fuzzing is no longer optional for serious smart contract security.
Unit tests validate expectations.
Fuzzing challenges assumptions.

Trident brings powerful, invariant-driven fuzzing to the Solana ecosystem in a practical and ergonomic way.

### The key takeaway

Fuzzing is not about randomness. It’s about modeling adversarial state transitions.
If you’re building or auditing Solana protocols, learning Trident will significantly increase the depth of your security testing and your ability to uncover subtle, high-impact bugs.