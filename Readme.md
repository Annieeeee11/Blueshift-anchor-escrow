# Blueshift Anchor Escrow

This is a Solana Anchor program that implements an escrow system for trustless token swaps. I learned this from Blueshift, and I'm documenting what I learned along the way. [Blueshift challenge link](https://learn.blueshift.gg/en/challenges/anchor-escrow)

## About

An escrow program that allows users to:

- **Make**: Create an escrow offer by depositing Token A and specifying how much Token B they want in return
- **Take**: Accept someone's escrow offer by transferring Token B to the maker and receiving Token A
- **Refund**: Cancel your own escrow and get your Token A back

The key thing I learned is that this creates a trustless swap neither party needs to trust the other because the smart contract enforces the rules automatically. It's like a digital safe deposit box where the terms are locked in code!

## What I Learned About PDAs and Escrow Accounts

At first, I was confused about how the escrow account could "own" tokens. I learned that we use a Program Derived Address (PDA) for the escrow account, which can sign transactions on behalf of the program.

In my program, the escrow PDA is created using:

- The string `"escrow"`
- The maker's public key
- A unique seed value (so one maker can create multiple escrows)

This was my first time using multiple seeds for a PDA! The seed parameter lets the same maker create different escrows, even with the same token pair. The bump byte is automatically found by Anchor and stored in the account to save compute units later.

The cool part is that the escrow PDA can "sign" transactions using these seeds, which is how we transfer tokens out of the vault during `take` and `refund` operations.

## Understanding the Account Structure

I learned that in Anchor, you define accounts using structs with constraints. This program has three main instruction structs, each with different accounts.

### Make Instruction

The `Make` struct has accounts for:
- `maker` - The user creating the escrow (marked as `mut` and `Signer`)
- `escrow` - The PDA escrow account (initialized with `init`)
- `mint_a` and `mint_b` - The token mints for both sides of the swap
- `maker_ata_a` - The maker's token account for Token A (where they deposit from)
- `vault` - A token account owned by the escrow PDA (where Token A is locked)
- Various programs (token program, associated token program, system program)

The `seeds` constraint on the escrow was tricky at first. I learned that:
- `seeds` tells Anchor how to derive the PDA address
- `bump` is automatically found and stored by Anchor
- The `init` constraint creates the account if it doesn't exist

### Take Instruction

The `Take` struct was more complex because it needs to:
- Validate the escrow exists and matches the provided mints
- Create ATAs for both the taker and maker if they don't exist
- Transfer tokens in both directions

I learned about the `has_one` constraint here checks that the escrow's stored values match the provided accounts. This prevents someone from trying to take an escrow with wrong token mints!

### Refund Instruction

The `Refund` struct is simpler, only the maker can refund their own escrow. The `has_one = maker` constraint ensures only the original maker can call this.

## Make Function

Here's what I learned while implementing `make`:

**Validate amounts** - I use `require_gte!` to make sure both `receive` and `amount` are valid (greater than or equal to 0). This was my first time using Anchor's validation macros.

**Populate the escrow** - I learned to use `set_inner()` to set all the fields of the escrow account at once. We store:
- The seed (so we can re-derive the PDA later)
- The maker's public key
- Both mint addresses
- How much Token B the maker wants (`receive`)
- The bump byte

**Deposit tokens** - This was my first time using SPL token transfers! I learned that:
- `transfer_checked` is used for token transfers (the `_checked` version validates decimals)
- We transfer from the maker's ATA to the vault
- The vault is owned by the escrow PDA, so the tokens are "locked" until someone takes the offer or the maker refunds

The `associated_token::` constraints were new to me. They automatically validate that the token account is an ATA for the specified mint and authority.

## Take Function

I had to learn about PDA signing for token transfers:

**Transfer Token B to maker** - The taker sends the required amount of Token B to the maker's ATA. I learned to use `init_if_needed` for the maker's ATA, this creates it if it doesn't exist, which is important because the maker might not have received this token type before.

**Withdraw Token A from vault** - This was the tricky part! I learned that:
- The vault is owned by the escrow PDA, not a regular keypair
- To transfer from it, I need to create signer seeds that prove I'm allowed to sign for this PDA
- The signer seeds must match exactly how the PDA was created

I learned that `to_le_bytes()` converts the seed to bytes, which is how it was stored during creation. The bump byte at the end is crucial without it, the signature would be invalid

**Close the vault** - After transferring all tokens out, I learned to use `close_account` to close the token account. This returns the rent to the maker (who paid for it initially). The `close = maker` constraint on the escrow account does the same thing, returns rent when the escrow is closed.

**Close the escrow** - The escrow account itself is also closed, returning rent to the maker. This was my first time seeing the `close` constraint in action!

## Refund Function

The refund function is similar to the withdrawal part of `take`, but simpler:

**Withdraw and close vault** - The maker gets their Token A back using the same PDA signing mechanism. I learned that even though the maker is the one calling this, they still need the PDA to sign because the vault is owned by the PDA, not the maker directly.

**Close the escrow** - Same as in `take`, the escrow account is closed and rent is returned.

The key learning here was understanding that ownership matters, even though the maker created the escrow, the tokens are technically owned by the PDA, so we need PDA signatures to move them.

## Understanding SPL Tokens and ATAs

This was my first deep dive into SPL tokens! I learned:

**Associated Token Accounts (ATAs)** - These are special token accounts that are deterministically derived from a wallet and mint. The `associated_token::` constraints in Anchor automatically validate and create these.

**Token Vaults** - The vault is just a regular token account, but it's owned by the escrow PDA instead of a user. This is how we "lock" the tokens only the PDA (which only our program can control) can move them.

**Token Transfers** - I learned the difference between `transfer` and `transfer_checked`:
- `transfer_checked` validates the decimals match, which prevents errors
- Both require the authority (owner) to sign the transaction

**Closing Token Accounts** - When you close a token account, you get the rent back. This is important for user experience, we don't want to leave empty accounts that cost rent

## Error Handling

I learned about Anchor's error system. Created a custom error enum with four errors:

- `InvalidAmount` - When the amount is invalid (negative or zero)
- `InvalidMaker` - When someone tries to take/refund an escrow they didn't create
- `InvalidMintA` - When the provided mint doesn't match the escrow's stored mint_a
- `InvalidMintB` - When the provided mint doesn't match the escrow's stored mint_b

The `#[error_code]` attribute tells Anchor to generate error codes automatically, and `#[msg()]` sets the error message. I learned that these errors are used by the `has_one` constraints and `require!` macros throughout the program.

## Running the Program

To build:

```bash
anchor build
```
