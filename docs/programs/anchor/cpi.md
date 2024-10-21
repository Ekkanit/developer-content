---
title: CPIs with Anchor
description:
  Learn how to implement Cross Program Invocations (CPIs) in Anchor programs,
  enabling interaction between different programs on Solana
sidebarLabel: CPIs with Anchor
sidebarSortOrder: 5
---

[Cross Program Invocations (CPI)](/docs/core/cpi.md) refer to the process of one
program invoking instructions of another program, which enables the
composibility of programs on Solana.

This section will cover the basics of implementing CPIs in an Anchor program,
using a simple SOL transfer instruction as a practical example. Once you
understand the basics of how to implement a CPI, you can apply the same concepts
for any instruction.

## Cross Program Invocations

Let's examine a program that implements a CPI to the System Program's transfer
instruction. Here is the example program on
[Solana Playground](https://beta.solpg.io/66df2751cffcf4b13384d35a).

The `lib.rs` file includes a single `sol_transfer` instruction. When the
`sol_transfer` instruction on the Anchor program is invoked, the program
internally invokes the transfer instruction of the System Program.

```rs filename="lib.rs" /sol_transfer/ /transfer/ {23}
use anchor_lang::prelude::*;
use anchor_lang::system_program::{transfer, Transfer};

declare_id!("9AvUNHjxscdkiKQ8tUn12QCMXtcnbR9BVGq3ULNzFMRi");

#[program]
pub mod cpi {
    use super::*;

    pub fn sol_transfer(ctx: Context<SolTransfer>, amount: u64) -> Result<()> {
        let from_pubkey = ctx.accounts.sender.to_account_info();
        let to_pubkey = ctx.accounts.recipient.to_account_info();
        let program_id = ctx.accounts.system_program.to_account_info();

        let cpi_context = CpiContext::new(
            program_id,
            Transfer {
                from: from_pubkey,
                to: to_pubkey,
            },
        );

        transfer(cpi_context, amount)?;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct SolTransfer<'info> {
    #[account(mut)]
    sender: Signer<'info>,
    #[account(mut)]
    recipient: SystemAccount<'info>,
    system_program: Program<'info, System>,
}
```

The `cpi.test.ts` file shows how to invoke the Anchor program's `sol_transfer`
instruction and logs a link to the transaction details on SolanaFM.

```ts filename="cpi.test.ts"
it("SOL Transfer Anchor", async () => {
  const transactionSignature = await program.methods
    .solTransfer(new BN(transferAmount))
    .accounts({
      sender: sender.publicKey,
      recipient: recipient.publicKey,
    })
    .rpc();

  console.log(
    `\nTransaction Signature:` +
      `https://solana.fm/tx/${transactionSignature}?cluster=devnet-solana`,
  );
});
```

You can build, deploy, and run the test for this example on Playground to view
the transaction details on the [SolanaFM explorer](https://solana.fm/).

The transaction details will show that the Anchor program was first invoked
(instruction 1), which then invokes the System Program (instruction 1.1),
resulting in a successful SOL transfer.

![Transaction Details](/assets/docs/core/cpi/transaction-details.png)

### Example 1 Explanation

Implementing a CPI follows the same pattern as building an instruction to add to
a transaction. When implementing a CPI, we must specify the program ID,
accounts, and instruction data for the instruction being called.

The System Program's transfer instruction requires two accounts:

- `from`: The account sending SOL.
- `to`: The account receiving SOL.

In the example program, the `SolTransfer` struct specifies the accounts required
by the transfer instruction. The System Program is also included because the CPI
invokes the System Program.

```rust /sender/ /recipient/ /system_program/
#[derive(Accounts)]
pub struct SolTransfer<'info> {
    #[account(mut)]
    sender: Signer<'info>, // from account
    #[account(mut)]
    recipient: SystemAccount<'info>, // to account
    system_program: Program<'info, System>, // program ID
}
```

The following tabs present three approaches to implementing Cross Program
Invocations (CPIs), each at a different level of abstraction. All examples are
functionally equivalent. The main purpose is to illustrate the implementation
details of the CPI.

<!-- prettier-ignore -->
<Tabs groupId="language" items={['1', '2', '3']}>
<Tab value="1">

The `sol_transfer` instruction included in the example code shows a typical
approach for constructing CPIs using the Anchor framework.

This approach involves creating a
[`CpiContext`](https://docs.rs/anchor-lang/latest/anchor_lang/context/struct.CpiContext.html),
which includes the `program_id` and accounts required for the instruction being
called, followed by a helper function (`transfer`) to invoke a specific
instruction.

```rust
use anchor_lang::system_program::{transfer, Transfer};
```

```rust /cpi_context/ {14}
pub fn sol_transfer(ctx: Context<SolTransfer>, amount: u64) -> Result<()> {
    let from_pubkey = ctx.accounts.sender.to_account_info();
    let to_pubkey = ctx.accounts.recipient.to_account_info();
    let program_id = ctx.accounts.system_program.to_account_info();

    let cpi_context = CpiContext::new(
        program_id,
        Transfer {
            from: from_pubkey,
            to: to_pubkey,
        },
    );

    transfer(cpi_context, amount)?;
    Ok(())
}
```

The `cpi_context` variable specifies the program ID (System Program) and
accounts (sender and recipient) required by the transfer instruction.

```rust /program_id/ /from_pubkey/ /to_pubkey/
let cpi_context = CpiContext::new(
    program_id,
    Transfer {
        from: from_pubkey,
        to: to_pubkey,
    },
);
```

The `cpi_context` and `amount` are then passed into the `transfer` function to
execute the CPI invoking the transfer instruction of the System Program.

```rust
transfer(cpi_context, amount)?;
```

</Tab>
<Tab value="2">

This example shows a different approach to implementing a CPI using the `invoke`
function and
[`system_instruction::transfer`](https://github.com/solana-labs/solana/blob/27eff8408b7223bb3c4ab70523f8a8dca3ca6645/sdk/program/src/system_instruction.rs#L881),
which is generally seen in native Rust programs.

Under the hood, the previous example is an abstraction of this implementation.
The example below is functionally equivalent to the previous example.

```rust
use anchor_lang::solana_program::{program::invoke, system_instruction};
```

```rust /instruction/1,3 {9}
pub fn sol_transfer(ctx: Context<SolTransfer>, amount: u64) -> Result<()> {
    let from_pubkey = ctx.accounts.sender.to_account_info();
    let to_pubkey = ctx.accounts.recipient.to_account_info();
    let program_id = ctx.accounts.system_program.to_account_info();

    let instruction =
        &system_instruction::transfer(&from_pubkey.key(), &to_pubkey.key(), amount);

    invoke(instruction, &[from_pubkey, to_pubkey, program_id])?;
    Ok(())
}
```

</Tab>
<Tab value="3">

You can also manually build the instruction to pass into the `invoke()`
function. This is useful when there is no crate available to help build the
instruction you want to invoke. This approach requires you to specify the
`AccountMeta`s for the instruction and correctly create the instruction data
buffer.

The `sol_transfer` instruction below is a manual implementation of a CPI to the
System Program's transfer instruction.

```rust /instruction/10,13 {28}
pub fn sol_transfer(ctx: Context<SolTransfer>, amount: u64) -> Result<()> {
    let from_pubkey = ctx.accounts.sender.to_account_info();
    let to_pubkey = ctx.accounts.recipient.to_account_info();
    let program_id = ctx.accounts.system_program.to_account_info();

    // Prepare instruction AccountMetas
    let account_metas = vec![
        AccountMeta::new(from_pubkey.key(), true),
        AccountMeta::new(to_pubkey.key(), false),
    ];

    // SOL transfer instruction discriminator
    let instruction_discriminator: u32 = 2;

    // Prepare instruction data
    let mut instruction_data = Vec::with_capacity(4 + 8);
    instruction_data.extend_from_slice(&instruction_discriminator.to_le_bytes());
    instruction_data.extend_from_slice(&amount.to_le_bytes());

    // Create instruction
    let instruction = Instruction {
        program_id: program_id.key(),
        accounts: account_metas,
        data: instruction_data,
    };

    // Invoke instruction
    invoke(&instruction, &[from_pubkey, to_pubkey, program_id])?;
    Ok(())
}
```

The `sol_transfer` instruction above replicates this
[example](/docs/core/transactions.md#manual-sol-transfer) of manually building a
SOL transfer instruction. It follows the same pattern as building an
[instruction](/docs/core/transactions.md#instruction) to add to a transaction.

When building an instruction in Rust, use the following syntax to specify the
`AccountMeta` for each account:

```rust
AccountMeta::new(account1_pubkey, true),           // writable, signer
AccountMeta::new(account2_pubkey, false),          // writable, not signer
AccountMeta::new_readonly(account3_pubkey, false), // not writable, not signer
AccountMeta::new_readonly(account4_pubkey, true),  // writable, signer
```

</Tab>
</Tabs>

Here is a reference program on
[Solana Playground](https://beta.solpg.io/github.com/ZYJLiu/doc-examples/tree/main/cpi)
which includes all 3 examples.

## Cross Program Invocations with PDA Signers

Next, let's examine a program that implements a CPI to the System Program's
transfer instruction where the sender is a Program Derived Address (PDA) that
must be "signed" for by the program. Here is the example program on
[Solana Playground](https://beta.solpg.io/66df2bd2cffcf4b13384d35b).

The `lib.rs` file includes the following program with a single `sol_transfer`
instruction.

```rust filename="lib.rs"
use anchor_lang::prelude::*;
use anchor_lang::system_program::{transfer, Transfer};

declare_id!("3455LkCS85a4aYmSeNbRrJsduNQfYRY82A7eCD3yQfyR");

#[program]
pub mod cpi {
    use super::*;

    pub fn sol_transfer(ctx: Context<SolTransfer>, amount: u64) -> Result<()> {
        let from_pubkey = ctx.accounts.pda_account.to_account_info();
        let to_pubkey = ctx.accounts.recipient.to_account_info();
        let program_id = ctx.accounts.system_program.to_account_info();

        let seed = to_pubkey.key();
        let bump_seed = ctx.bumps.pda_account;
        let signer_seeds: &[&[&[u8]]] = &[&[b"pda", seed.as_ref(), &[bump_seed]]];

        let cpi_context = CpiContext::new(
            program_id,
            Transfer {
                from: from_pubkey,
                to: to_pubkey,
            },
        )
        .with_signer(signer_seeds);

        transfer(cpi_context, amount)?;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct SolTransfer<'info> {
    #[account(
        mut,
        seeds = [b"pda", recipient.key().as_ref()],
        bump,
    )]
    pda_account: SystemAccount<'info>,
    #[account(mut)]
    recipient: SystemAccount<'info>,
    system_program: Program<'info, System>,
}
```

The `cpi.test.ts` file shows how to invoke the Anchor program's `sol_transfer`
instruction and logs a link to the transaction details on SolanaFM.

It shows how to derive the PDA using the seeds specified in the program:

```ts /pda/ /wallet.publicKey/
const [PDA] = PublicKey.findProgramAddressSync(
  [Buffer.from("pda"), wallet.publicKey.toBuffer()],
  program.programId,
);
```

The first step in this example is to fund the PDA account with a basic SOL
transfer from the Playground wallet.

```ts filename="cpi.test.ts"
it("Fund PDA with SOL", async () => {
  const transferInstruction = SystemProgram.transfer({
    fromPubkey: wallet.publicKey,
    toPubkey: PDA,
    lamports: transferAmount,
  });

  const transaction = new Transaction().add(transferInstruction);

  const transactionSignature = await sendAndConfirmTransaction(
    connection,
    transaction,
    [wallet.payer], // signer
  );

  console.log(
    `\nTransaction Signature:` +
      `https://solana.fm/tx/${transactionSignature}?cluster=devnet-solana`,
  );
});
```

Once the PDA is funded with SOL, invoke the `sol_transfer` instruction. This
instruction transfers SOL from the PDA account back to the `wallet` account via
a CPI to the System Program, which is "signed" for by the program.

```ts
it("SOL Transfer with PDA signer", async () => {
  const transactionSignature = await program.methods
    .solTransfer(new BN(transferAmount))
    .accounts({
      pdaAccount: PDA,
      recipient: wallet.publicKey,
    })
    .rpc();

  console.log(
    `\nTransaction Signature: https://solana.fm/tx/${transactionSignature}?cluster=devnet-solana`,
  );
});
```

You can build, deploy, and run the test to view the transaction details on the
[SolanaFM explorer](https://solana.fm/).

The transaction details will show that the custom program was first invoked
(instruction 1), which then invokes the System Program (instruction 1.1),
resulting in a successful SOL transfer.

![Transaction Details](/assets/docs/core/cpi/transaction-details-pda.png)

### Example 2 Explanation

In the example code, the `SolTransfer` struct specifies the accounts required by
the transfer instruction.

The sender is a PDA that the program must sign for. The `seeds` to derive the
address for the `pda_account` include the hardcoded string "pda" and the address
of the `recipient` account. This means the address for the `pda_account` is
unique for each `recipient`.

```rust /pda_account/ /recipient/2 /system_program/
#[derive(Accounts)]
pub struct SolTransfer<'info> {
    #[account(
        mut,
        seeds = [b"pda", recipient.key().as_ref()],
        bump,
    )]
    pda_account: SystemAccount<'info>,
    #[account(mut)]
    recipient: SystemAccount<'info>,
    system_program: Program<'info, System>,
}
```

The Javascript equivalent to derive the PDA is included in the test file.

```ts /pda/ /wallet.publicKey/
const [PDA] = PublicKey.findProgramAddressSync(
  [Buffer.from("pda"), wallet.publicKey.toBuffer()],
  program.programId,
);
```

The following tabs present two approaches to implementing Cross Program
Invocations (CPIs), each at a different level of abstraction. Both examples are
functionally equivalent. The main purpose is to illustrate the implementation
details of the CPI.

<!-- prettier-ignore -->
<Tabs groupId="language" items={['1', '2']}>
<Tab value="1">

The `sol_transfer` instruction included in the example code shows a typical
approach for constructing CPIs using the Anchor framework.

This approach involves creating a
[`CpiContext`](https://docs.rs/anchor-lang/latest/anchor_lang/context/struct.CpiContext.html),
which includes the `program_id` and accounts required for the instruction being
called, followed by a helper function (`transfer`) to invoke a specific
instruction.

```rust /cpi_context/ {19}
pub fn sol_transfer(ctx: Context<SolTransfer>, amount: u64) -> Result<()> {
    let from_pubkey = ctx.accounts.pda_account.to_account_info();
    let to_pubkey = ctx.accounts.recipient.to_account_info();
    let program_id = ctx.accounts.system_program.to_account_info();

    let seed = to_pubkey.key();
    let bump_seed = ctx.bumps.pda_account;
    let signer_seeds: &[&[&[u8]]] = &[&[b"pda", seed.as_ref(), &[bump_seed]]];

    let cpi_context = CpiContext::new(
        program_id,
        Transfer {
            from: from_pubkey,
            to: to_pubkey,
        },
    )
    .with_signer(signer_seeds);

    transfer(cpi_context, amount)?;
    Ok(())
}
```

When signing with PDAs, the seeds and bump seed are included in the
`cpi_context` as `signer_seeds` using `with_signer()`. The bump seed for a PDA
can be accessed using `ctx.bumps` followed by the name of the PDA account.

```rust /signer_seeds/ /bump_seed/ {3}
let seed = to_pubkey.key();
let bump_seed = ctx.bumps.pda_account;
let signer_seeds: &[&[&[u8]]] = &[&[b"pda", seed.as_ref(), &[bump_seed]]];

let cpi_context = CpiContext::new(
    program_id,
    Transfer {
        from: from_pubkey,
        to: to_pubkey,
    },
)
.with_signer(signer_seeds);
```

The `cpi_context` and `amount` are then passed into the `transfer` function to
execute the CPI.

```rust
transfer(cpi_context, amount)?;
```

When the CPI is processed, the Solana runtime will validate that the provided
seeds and caller program ID derive a valid PDA. The PDA is then added as a
signer on the invocation. This mechanism allows for programs to sign for PDAs
that are derived from their program ID.

</Tab>
<Tab value="2">

Under the hood, the previous example is a wrapper around the `invoke_signed()`
function which uses
[`system_instruction::transfer`](https://github.com/solana-labs/solana/blob/27eff8408b7223bb3c4ab70523f8a8dca3ca6645/sdk/program/src/system_instruction.rs#L881)
to build the instruction.

This example shows how to use the `invoke_signed()` function to make a CPI
signed for by a PDA.

```rust
use anchor_lang::solana_program::{program::invoke_signed, system_instruction};
```

```rust /instruction/1,3 {13}
pub fn sol_transfer(ctx: Context<SolTransfer>, amount: u64) -> Result<()> {
    let from_pubkey = ctx.accounts.pda_account.to_account_info();
    let to_pubkey = ctx.accounts.recipient.to_account_info();
    let program_id = ctx.accounts.system_program.to_account_info();

    let seed = to_pubkey.key();
    let bump_seed = ctx.bumps.pda_account;
    let signer_seeds: &[&[&[u8]]] = &[&[b"pda", seed.as_ref(), &[bump_seed]]];

    let instruction =
        &system_instruction::transfer(&from_pubkey.key(), &to_pubkey.key(), amount);

    invoke_signed(instruction, &[from_pubkey, to_pubkey, program_id], signer_seeds)?;
    Ok(())
}
```

This implementation is functionally equivalent to the previous example. The
`signer_seeds` are passed into the `invoke_signed` function.

</Tab>
</Tabs>

Here is a reference program on
[Solana Playground](https://beta.solpg.io/github.com/ZYJLiu/doc-examples/tree/main/cpi-pda)
which includes both examples.