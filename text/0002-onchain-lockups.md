- Start Date: 2020-10-13
- RFC PR: N/A
- Issue: N/A

# Summary
[summary]: #summary

Defines the Serum Lockup program.

# Design
[design]: #design

## Introduction

The lockup program does two basic things: 1) enforce access control on the dispersion of
a funded SRM vesting account according to the account's vesting schedule and 2)
allows the productive use of funds locked in a vesting account by governing a whitelist
of programs that beneficiaries of vesting accounts are allowed to send their locked SRM
back and forth to.

For example, one can use the Lockup program to receive a locked SRM grant that vests over
7 years, stake that same SRM immediately, and then withdraw any of it as dictated by the
rules of the vesting schedule--in addition to earning staking rewards while the SRM
was locked up.

## Accounts

The lockup program owns three accounts; `Safe`, `Vesting`, and `Whitelist`.

### Safe

The `Safe` account represents an instance of the program.

```rust
struct Safe {
    /// Is `true` if this structure has been initialized
    initialized: bool,
    /// The mint of the SPL token the safe is storing, e.g., the SRM mint.
    mint: Pubkey,
    /// The key with the ability to migrate or change the authority.
    authority: Pubkey,
    /// The nonce to use for the program-derived-address owning the Safe's
    /// token vault.
    nonce: u8,
    /// The whitelist of valid programs the Safe can relay transactions to.
    whitelist: Pubkey,
    /// Address of the token vault controlled by the Safe.
    vault: Pubkey,
}
```

Most notably, this account has a program controlled token vault locking all SRM
and is subject to relase by the parameters defined in a separate `Vesting` account.
In addition, an `authority` key exists to support a priviledged set of instructions,
invokable by the `authority` as signer. This key can be owned by a single account or governed by, for example,
a multisig program. The program doesn't care.

### Vesting

The `Vesting` account represents a deposit allocated on behalf of a `beneficiary`.

```rust
struct Vesting {
    /// True iff the vesting account has been initialized via deposit.
    initialized: bool,
    /// One time token for claiming the vesting account.
    claimed: bool,
    /// The Safe instance this account is associated with.
    safe: Pubkey,
    /// The effective owner of this Vesting account. I.e., the only one
    /// that can withdraw and use locked SRM.
    beneficiary: Pubkey,
    /// The outstanding SRM deposit backing this vesting account. All
    /// withdrawals/redemptions will deduct this balance.
    balance: u64,
    /// The starting balance of this vesting account, i.e., how much was
    /// originally deposited.
    start_balance: u64,
    /// The slot at which this vesting account was created.
    start_slot: u64,
    /// The slot at which all the tokens associated with this account
    /// should be vested.
    end_slot: u64,
    /// The number of times vesting will occur. For example, if vesting
    /// is once a year over seven years, this will be 7.
    period_count: u64,
    /// The spl token mint associated with this vesting account. The supply
    /// should always equal the `balance` field.
    locked_nft_mint: Pubkey,
    /// The token account the locked_nft supply is expected to be in.
    locked_nft_token: Pubkey,
    /// The amount of tokens in custody of whitelisted programs. As this number
    /// goes up, the *vested* amount available for withdrawal, goes down.
    whitelist_owned: u64,
}
```

Note the fields defining a linear vesting schedule: `start_slot`, `end_slot`, `period_count`, and `deposit_amount`.
In order to conveniently parititon the vesting schedule
into `period_count` chunks, one can expect the first vesting period to act somewhat as a "cliff". That is, suppose a
vesting account was created at `start_slot`, then if `(end_slot - start_slot) % period_count != 0`) then we can
pretend `start_slot = start_slot - (end_slot - start+slot) % period_count` , giving us a perfectly partitioned
vesting schedule. This has the effect of making the first vesting time period potentially shorter than the rest.

Additionally, note the `locked_nft_mint` and `locked_nft_token` fields. These are used to define an SPL token
used to represent an effective "receipt" from the `Lockup` program. This SPL token can be used, not only to view
your `Vesting` account `balance`, but it is also required as a redemption token, to exchange for the locked SRM
upon vesting.

### Whitelist

The whitelist represents a list of all valid--and *trusted*--programs to send locked SRM to.
These programs are completely trusted, so if there's a bug in one, locked SRM can escape and
vest before the intended time period. As a result, it's critical these whitelisted programs
are governed, managed, and audited by the community so that they don't lead to undesired behavior.

```rust
struct Whitelist {
  entries: [WhitelistEntry; 50];
}
```
Each `WhitelistEntry` in the `Whitelist` is *effectively* a three element tuple used to derive a program
controlled address, which is expected to be the authority on a token vault controlled by the
program. The three tuple fields represent the `program_id`, `instance`, and `nonce` of a program,
where the latter two fields are used as the "seed" to the address derivation. `instance`,
can be defined in any way by the targeted program and `nonce` is used as the additional seed.
For example, if one wanted to add the `Lockup` program itself to the `Whitelist` (though there's no reason to),
one would use the `Safe` account's `Pubkey` address as the `instance`along with it's nonce to define the
`WhitelistEntry` field, which ultimately allows for the derivation of the program-controlledaddress owning
its SRM vault.

For more on how the whitelist is used, see the `WhitelistWithdrawal`s and `WhitelistDeposit`s instructions

## Initialization and Governance Instructions

### Initialize Instruction

After deploying the `Lockup` program on chain, the `Safe` must be initialized with an initial configuration.

```rust
/// Accounts:
///
/// 0. `[writable]` Safe to initialize.
/// 1. `[writable]` Whitelist to initialize.
/// 2. `[]`         Vault.
/// 4. `[]`         Mint of the SPL token controlled by the safe.
/// 5. `[]`         Rent sysvar.
Initialize {
    /// The priviledged account whose signature is required for governance instructions.
    authority: Pubkey,
    /// The nonce to use to create the Safe's derived-program address,
    /// which is used as the authority for the safe's token vault.
    nonce: u8,
}
```
Namely, the `Whitelist` address, `mint` to lockup, `authority` to govern, `vault` holding tokens, and `nonce` defining the
vault's owner must be given.

### Adding and Removing from the Whitelist

```rust
/// Accounts:
///
/// 0. `[signed]`   Safe authority.
/// 1. `[]`         Safe account.
/// 2. `[writable]` Whitelist.
WhitelistAdd {
    entry: WhitelistEntry,
}

/// Accounts:
///
/// 0. `[signed]`   Safe authority.
/// 1. `[]`         Safe account.
/// 2. `[writable]` Whitelist.
WhitelistDelete {
    entry: WhitelistEntry,
}
```

Where `WhitelistEntry` is as defined in the Accounts section.

### Setting a new Authority

To change the priviledged account, invoke the `SetAuthority` instruction. It's expected this will
be used to transition the `Lockup` program to a community governed program after the `Lockup`
program and `Safe` has been properly initialized.

```rust
SetAuthority { new_authority: Pubkey }
```

## Creating a Vesting Account

A vesting account is created upon the `CreateVesting` instruction that anyone can call.

```rust
/// Accounts:
///
/// 0. `[writable]  Vesting account representing this deposit.
/// 1. `[writable]` Depositor token account, transferring ownership
///                 *from*, itself to this program's vault.
/// 2. `[signer]`   The authority||owner||delegate of Accounts[1].
/// 3. `[writable]` The program controlled token vault, transferring
///                 ownership *to*. The owner of this account is the
///                 program derived address with nonce set in the
///                 Initialize instruction.
/// 4. `[]`         Safe instance.
/// 5. `[writable]` Token mint representing the locked SRM receipt.
/// 6. `[]`         Safe's vault authority, a program derived address.
///                 The mint authority.
/// 7. `[]`         SPL token program.
/// 8. `[]`         Rent sysvar.
/// 9. `[]`         Clock sysvar.
CreateVesting {
    beneficiary: Pubkey,
    end_slot: u64,
    period_count: u64,
    deposit_amount: u64,
}
```

The parameters here are as described in the `Vesting` account section.

## Claiming a Vesting Account

Once a `Vesting` account has been created on behalf of a `beneficiary`, the `beneficiary` must invoke the `Claim`
instruction before it can do anything with the vesting account.

```rust
/// Accounts:
///
/// 0. `[signer]`   Vesting account beneficiary.
/// 1. `[writable]` Vesting account.
/// 2. `[]`         Safe instance.
/// 3. `[]`         Safe's vault authority, a program derived address.
/// 4. `[]`         SPL token program.
/// 5. `[writable]` Token mint representing the lSRM receipt.
/// 6  `[writable]` Token account associated with the mint.
Claim
```

This instruction can be invoked exactly once, minting the Vesting account's `locked_nft_mint` (see the `Vesting` account
section for a descrioption) to the given `locked_nft_token` given in `Accounts[6]`. This token is required for
eventually redeeming the locked SRM upon vesting. Additionally, the `locked_nft_token`'s owner *must* be the `beneficiary`
for both `Claim` and the next instruction `Redeem`.

## Withdrawing Vested Tokens

To withdraw locked SRM upon vesting, invoke the `Redeem` instruction to exchange the "claimed" minted token
receipt for the underlying SRM.

```rust
/// Accounts:
///
/// 0. `[signer]`   Vesting account's beneficiary.
/// 1. `[writable]` Vesting account to withdraw from.
/// 2. `[writable]` SPL token account to withdraw to.
/// 3. `[writable]` Safe's token account vault from which we are
///                 transferring ownership of the SRM out of.
/// 4. `[]`         Safe's vault authority, i.e., the program-derived
///                 address.
/// 5  `[]`         Safe account.
/// 6. `[writable]` NFT token being redeemed.
/// 7. `[writable]` NFT mint to burn the token being redeemed.
/// 8. `[]`         SPL token program.
/// 9. `[]`         Clock sysvar.
Redeem { amount: u64 }
```

Not only must the beneficiary sign this transaction, but also the token and associated mint must be
as defined in the `Vesting` account, with the `beneficiary` as the owner of the token account.

## Using Locked Tokens

### Withdrawing from the Lockup Program to a Whitelisted Address

If one wants to, for example, stake locked SRM, one would have to invoke the `WithdrawWhitelist`
instruction, moving funds from the `Lockup` program to the staking program assuming it's on the
whitelist.

```rust
/// Accounts:
///
/// 0. `[signer]`   Vesting beneficiary.
/// 1. `[writable]` Vesting.
/// 2. `[]`         Safe (containing the nonce).
/// 3. `[]`         Safe vault authority.
/// 4. `[]`         Whitelisted program to invoke.
/// 5. `[]`         Whitelist.
///
/// All accounts below will be relayed to the whitelisted program.
///
/// 6. `[writable]` Safe vault.
/// 7. `[writable]` Vault which will receive funds.
/// 8. `[]`         Whitelisted vault authority.
/// 9. `[]`         Token program id.
/// .. `[writable]` Variable number of program specific accounts to
///                 relay to the program, along with the above
///                 whitelisted accounts and Safe vault.
WhitelistWithdraw {
    /// Amount of funds the whitelisted program is approved to
    /// transfer to itself. Must be less than or equal to the vesting
    /// account's whitelistable balance.
    amount: u64,
    /// Opaque instruction data to relay to the whitelisted program.
    instruction_data: Vec<u8>,
}
```

It is expected the whitelisted program implements a specific interface.

First it must accept all accounts in the range `6-9`, inclusive, in that order. And
all program specific accounts, must come after, as above. Furthermore, the instruction
provides an opaque `instruction_data` field, which it will relay to the targeted
program to interpret as desired. This means the client invoking this instruction is responsible
for properly serializing the Whitelisted program's instruction data and passing it in prior
to serializing the `WhitelistWithdraw` instruction itself.

Upon executing this instruction, the `Lockup` program will perform three steps:

* Check if the targeted program and it's associated vault authority is whitelisted
* Approve the targeted program's vault authority as a token delegate for the given `amount`
* Invoke the cross program invocation with the *relayed* accounts and `instruction_data`
* Revoke delegate access upon completion, recording the amount actually transferred to the whitelisted program in the `VestingAccount`'s `whitelist_owned` field

If successful the funds will have been transferred to the whitelisted program on behalf of the
beneficiary, and it's up to the beneficiary to invoke a `WhitelistDeposit` instruction to
return the funds to the `Lockup` program.

### Depositing to the Lockup Program from a Whitelisted Address

The `WhitelistDeposit` instruction works similarly to the `WhitelistWithdraw` instruction but in the opposite direction.
```rust
/// Accounts:
///
/// Same as WhitelistWithdraw.
WhitelistDeposit { instruction_data: Vec<u8> }
```
Specifically, it

* Records the balance of the vault of the Lockup program
* Makes a cross program invocation to the target whitelisted program
* Checks the new balance of the vault, and if it increased, records the balance increase on the `Vesting` account by decreasing its `whitelist_owned` field

Importantly, the cross program invocation is signed by the `Safe`'s program derived address, the signature of which
can be used for authorization in the targeted whitelisted program.

# Interface
[interface]: #interface

For completeness, the full interface is as follows.

```rust
enum LockupInstruction {
    /// Initializes a safe instance for use.
    ///
    /// Accounts:
    ///
    /// 0. `[writable]` Safe to initialize.
    /// 1. `[writable]` Whitelist to initialize.
    /// 2. `[]`         Mint of the SPL token controlled by the safe.
    /// 3. `[]`         Rent sysvar
    Initialize {
        /// The priviledged account.
        authority: Pubkey,
        /// The nonce to use to create the Safe's derived-program address,
        /// which is used as the authority for the safe's token vault.
        nonce: u8,
    },
    /// CreateVesting initializes a vesting account, transferring tokens
    /// from the controlling token account to one owned by the program
    /// Anyone with funds to deposit can invoke this instruction.
    ///
    /// Accounts:
    ///
    /// 0. `[writable]  Vesting account representing this deposit.
    /// 1. `[writable]` Depositor token account, transferring ownership
    ///                 *from*, itself to this program's vault.
    /// 2. `[signer]`   The authority||owner||delegate of Accounts[1].
    /// 3. `[writable]` The program controlled token vault, transferring
    ///                 ownership *to*. The owner of this account is the
    ///                 program derived address with nonce set in the
    ///                 Initialize instruction.
    /// 4. `[]`         Safe instance.
    /// 5. `[writable]` Token mint representing the lSRM receipt.
    /// 6. `[]`         Safe's vault authority, a program derived address.
    ///                 The mint authority.
    /// 7. `[]`         SPL token program.
    /// 8. `[]`         Rent sysvar.
    /// 9. `[]`         Clock sysvar.
    CreateVesting {
        /// The beneficiary of the vesting account, i.e.,
        /// the user who will own the SRM upon vesting.
        beneficiary: Pubkey,
        /// The Solana slot number at which point the entire deposit will
        /// be vested.
        end_slot: u64,
        /// The number of vesting periods for the account. For example,
        /// a vesting yearly over seven years would make this 7.
        period_count: u64,
        /// The amount to deposit into the vesting account.
        deposit_amount: u64,
    },
    /// Claim is an instruction for one time use by the beneficiary of a
    /// Vesting account. It mints a non-fungible SPL token and sends it
    /// to an account owned by the beneficiary as a of receipt SRM locked.
    ///
    /// The beneficiary, and only the beneficiary, can redeem this token
    /// in exchange for the underlying asset as soon as the account vests.
    ///
    /// Accounts:
    ///
    /// 0. `[signer]`   Vesting account beneficiary.
    /// 1. `[writable]` Vesting account.
    /// 2. `[]`         Safe instance.
    /// 3. `[]`         Safe's vault authority, a program derived address.
    /// 4. `[]`         SPL token program.
    /// 5. `[writable]` Token mint representing the lSRM receipt.
    /// 6  `[writable]` Token account associated with the mint.
    Claim,
    /// Reedeem exchanges the given `amount` of non-fungible, claimed
    /// receipt tokens for the underlying locked SRM, subject to the
    /// Vesting account's vesting schedule.
    ///
    /// Accounts:
    ///
    /// 0. `[signer]`   Vesting account's beneficiary.
    /// 1. `[writable]` Vesting account to withdraw from.
    /// 2. `[writable]` SPL token account to withdraw to.
    /// 3. `[writable]` Safe's token account vault from which we are
    ///                 transferring ownership of the SRM out of.
    /// 4. `[]`         Safe's vault authority, i.e., the program-derived
    ///                 address.
    /// 5  `[]`         Safe account.
    /// 6. `[writable]` NFT token being redeemed.
    /// 7. `[writable]` NFT mint to burn the token being redeemed.
    /// 8. `[]`         SPL token program.
    /// 9. `[]`         Clock sysvar.
    Redeem { amount: u64 },
    /// Invokes an opaque instruction on a whitelisted program,
    /// giving it delegate access to send `amount` funds to itself.
    ///
    /// For example, a user could call this with a staking program
    /// instruction to send locked SRM to it without custody ever leaving
    /// an on-chain program.
    ///
    /// Accounts:
    ///
    /// 0. `[signer]`   Vesting beneficiary.
    /// 1. `[writable]` Vesting.
    /// 2. `[]`         Safe (containing the nonce).
    /// 3. `[]`         Safe vault authority.
    /// 4. `[]`         Whitelisted program to invoke.
    /// 5. `[]`         Whitelist.
    ///
    /// All accounts below will be relayed to the whitelisted program.
    ///
    /// 6. `[writable]` Safe vault.
    /// 7. `[writable]` Vault which will receive funds.
    /// 8. `[]`         Whitelisted vault authority.
    /// 9. `[]`         Token program id.
    /// .. `[writable]` Variable number of program specific accounts to
    ///                 relay to the program, along with the above
    ///                 whitelisted accounts and Safe vault.
    WhitelistWithdraw {
        /// Amount of funds the whitelisted program is approved to
        /// transfer to itself. Must be less than or equal to the vesting
        /// account's whitelistable balance.
        amount: u64,
        /// Opaque instruction data to relay to the whitelisted program.
        instruction_data: Vec<u8>,
    },
    /// Makes an opaque cross program invocation to a whitelisted program.
    /// It's expected the CPI will deposit funds into the Safe's vault.
    ///
    /// Accounts:
    ///
    /// Same as WhitelistWithdraw.
    WhitelistDeposit { instruction_data: Vec<u8> },
    /// Adds the given entry to the whitelist.
    ///
    /// Accounts:
    ///
    /// 0. `[signed]`   Safe authority.
    /// 1. `[]`         Safe account.
    /// 2. `[writable]` Whitelist.
    WhitelistAdd {
        entry: WhitelistEntry,
    },
    /// Removes the given entry from the whitelist.
    ///
    /// Accounts:
    ///
    /// 0. `[signed]`   Safe authority.
    /// 1. `[]`         Safe account.
    /// 2. `[writable]` Whitelist.
    WhitelistDelete {
        entry: WhitelistEntry,
    },
    /// Sets the new authority for the safe instance.
    ///
    /// 0. `[signer]`   Current safe authority.
    /// 1. `[writable]` Safe instance.
    SetAuthority { new_authority: Pubkey },
}
```
