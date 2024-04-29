In this guide, you will learn how to create a react-native mobile app that is both android and ios compatible. This app will mimic a cash app experience but run on the solana blockchian, showcasing that web3 products can have the same user expereince as web2 products. To build this, we will need to write an anchor program with escrow functionality, integrate the solana name service sdk, and intergrate solana pay.

## What you will learn

- Setting up your environment
- Creating a solana mobile dApp
- Anchor program development
- Anchor PDAs and accounts
- Deploying a Solana program
- Testing an on-chain program
- Connecting an on-chain program to a mobile react-native UI

## Prerequisites

For this guide, you will need to have your local development environment setup
with a few tools:

- [Rust](https://www.rust-lang.org/tools/install)
- [Node JS](https://nodejs.org/en/download)
- [Solana CLI & Anchor](https://solana.com/developers/guides/getstarted/setup-local-development)
- [Android Studio and emmulator set up](https://docs.solanamobile.com/getting-started/development-setup)
- [React Native Setup](https://reactnative.dev/docs/environment-setup?platform=android)
- [EAS CLI and Account Setup](https://docs.expo.dev/build/setup/)

If you are new to solana program development, review this guide first:

- [Basic CRUD dApp on Solana](https://github.com/solana-foundation/developer-content/blob/main/content/guides/dapps/journal.md#writing-a-solana-program-with-anchor)

If you are new to mobile development, take a look at the solana mobile docs:

- [Solana Mobile Introduction](https://docs.solanamobile.com/getting-started/intro)

## Project design overview

Let's start by quickly mapping out the entire dApp design. To create a clone of cash app, we want to have the following functionalities:

1. Account Creation
2. Deposit and Withdrawal Funds
3. Peer-to-Peer Money Transfer
4. Payment Protection
5. QR Code Generation
6. Connect with Friends
7. Activity Tracking

To enable these functionalies, we will do the following:

1. Write a solana program that allows for users to initialize a new account on-chain and set up a user name _(similar to $Cashtag)_ with [Solana Name Service](https://sns.guide/). With the username being set via SNS, you can then get publickey information direclty from an account's username.
2. Add instructions to the solana program for a user to be able to deposit funds from their wallet into their cash account and withdrawal funds from their cash account into their wallet.
3. Add instructions for a user to be able to directly send funds from their own cash account to another cash account, request funds from a specified cash account, and accept or decline payment requests.
4. Create an escrow account within the program to be able to hold funds for a specifed period of time when requested by the user to enable payment protection. These payments will show as "pending" while they sit in the escrow allowing time for a user to revoke the payment if needed.
5. Integrate [Solana Pay](https://docs.solanapay.com/) to enable QR code generation. Solana pay also allows you to specify the amount and memo for the requested transaction directly in the QR code.
6. Update the solana program to save friends to your account state, which can then be displayed on the front end similar to cash app.
7. Add an activity tab to showcase pending requests, pending payments, and recent transactions.

## Setting up the project

Since this project will be a mobile app, we can get started with the solana mobile expo app template:

```shell
yarn create expo-app --template @solana-mobile/solana-mobile-expo-template
```

Name the project `cash-app-clone` then navigate into the directory.

Follow the [Running the app](https://docs.solanamobile.com/react-native/expo#running-the-app) guide to launch the template as a custom development build and get it running on your andriod emmulator. Once you have built the program and are running a dev client with expo, the emmulator will automatically update everytime you save your code.

Reminder: You must have [fake wallet](https://github.com/solana-mobile/mobile-wallet-adapter/tree/main/android/fakewallet) running on the same android emulator to be able to test out transactions, as explained in the [solana mobile development set up docs](https://docs.solanamobile.com/getting-started/development-setup).

## Section One: Cash App Basic Functionalities

We'll break up the solana program code into a few sections. To start off, lets just enable account creation, deposits, withdrawals, and direct transfers of funds.

In `cash-app-clone`, create a folder for your anchor program and a `lib.rs` file within that folder.

### Define your Anchor program

```rust
use anchor_lang::prelude::*;

declare_id!("7AGmMcgd1SjoMsCcXAAYwRgB9ihCyM8cZqjsUqriNRQt");

#[program]
pub mod cash_app {
    use super::*;
}
```

### Define your program state

To enable basic cash app functionalities for section one of this tutorial, we will want to save the following information to our cash account state:

```rust
#[account]
#[derive(InitSpace)]
pub struct CashAccount {
    pub balance: u64,
    pub owner: Pubkey,
    pub friends: Vec<Pubkey>,
}
```

### Adding instructions

Now that the state is defined, we need to create an instruction to initalize an account when a new user signs up for cash app. This will initialize a new account and save the user's public key into the PDA of the user's cash account.

```rust
#[program]
pub mod cash_app {
    use super::*;

    pub fn initialize_account(ctx: Context<InitializeAccount>) -> Result<()> {
        let cash_account = &mut ctx.accounts.cash_account;
        cash_account.balance = 0;
        cash_account.owner = *ctx.accounts.user.key;
        cash_account.friends = Vec::new();
        Ok(())
    }
}

#[derive(Accounts)]
pub struct InitializeAccount<'info> {
    #[account(
        init,
        seeds = [b"cash-account", user.key().as_ref()],
        bump,
        payer = user,
        space = 8 + CashAccount::INIT_SPACE
    )]
    pub cash_account: Account<'info, CashAccount>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}

```

Next we will need to add an instruction to this program that allows a user to deposit funds into their cash account:

```rust
#[program]
pub mod cash_app {
    use super::*;

    ...

    pub fn deposit_funds(ctx: Context<DepositFunds>, amount: u64) -> Result<()> {
        require!(amount > 0, ErrorCode::InvalidAmount);

        let ix = system_instruction::transfer(
            &ctx.accounts.user.key(),
            ctx.accounts.cash_account.to_account_info().key,
            amount,
        );

        invoke(
            &ix,
            &[
                ctx.accounts.user.clone(),
                ctx.accounts.cash_account.to_account_info(),
            ],
        )?;

        ctx.accounts.cash_account.balance = ctx
            .accounts
            .cash_account
            .balance
            .checked_add(amount)
            .ok_or(ErrorCode::Overflow)?;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct DepositFunds<'info> {
    #[account(mut)]
    pub cash_account: Account<'info, CashAccount>,
    #[account(mut)]
    /// CHECK: This account is only used to transfer SOL, not for data storage.
    pub user: AccountInfo<'info>,
    pub system_program: Program<'info, System>,
}

```

Now lets add an instruction into this program that allow a user to deposit funds into their cash account:

```rust
#[program]
pub mod cash_app {
    use super::*;

    ...

    pub fn withdraw_funds(ctx: Context<WithdrawFunds>, amount: u64) -> Result<()> {
        require!(amount > 0, ErrorCode::InvalidAmount);

        let cash_account = ctx.accounts.cash_account.to_account_info();
        let wallet = ctx.accounts.user.to_account_info();

        **cash_account.try_borrow_mut_lamports()? -= amount;
        **wallet.try_borrow_mut_lamports()? += amount;

        ctx.accounts.cash_account.balance = ctx
            .accounts
            .cash_account
            .balance
            .checked_sub(amount)
            .ok_or(ErrorCode::Overflow)?;

        Ok(())
    }
}

#[derive(Accounts)]
pub struct WithdrawFunds<'info> {
    #[account(
        mut,
        seeds = [b"cash-account", user.key().as_ref()],
        bump,
    )]
    pub cash_account: Account<'info, CashAccount>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}

```

The last instruction needed to enable the basic functionalities defined is to be able to directly transfer funds from one user to another.

```rust
#[program]
pub mod cash_app {
    use super::*;

    ...

    pub fn transfer_funds(ctx: Context<TransferFunds>, amount: u64) -> Result<()> {
        require!(amount > 0, ErrorCode::InvalidAmount);
        let to_cash_account = &mut ctx.accounts.to_cash_account;
        let from_cash_account = &mut ctx.accounts.from_cash_account;

        if from_cash_account.balance < amount {
            return Err(ErrorCode::InsufficientFunds.into());
        }

        let sender = from_cash_account.clone().to_account_info();
        let recipient = to_cash_account.clone().to_account_info();

        **sender.try_borrow_mut_lamports()? -= amount;
        **recipient.try_borrow_mut_lamports()? += amount;

        from_cash_account.balance = from_cash_account
            .balance
            .checked_sub(amount)
            .ok_or(ErrorCode::Overflow)?;
        to_cash_account.balance = to_cash_account
            .balance
            .checked_add(amount)
            .ok_or(ErrorCode::Overflow)?;

        Ok(())
    }
}

#[derive(Accounts)]
pub struct TransferFunds<'info> {
    #[account(
        mut,
        seeds = [b"cash-account", user.key().as_ref()],
        bump,
    )]
    pub from_cash_account: Account<'info, CashAccount>,
    #[account(mut)]
    pub to_cash_account: Account<'info, CashAccount>,
    pub system_program: Program<'info, System>,
    // The owner of the from_cash_account must sign the transaction
    pub user: Signer<'info>,
}
```

Note: If there is anyh confusion on the above anchor macros or structs defined for the instruiciton context, please refer to the [Basic CRUD dApp on Solana Guide.](https://github.com/solana-foundation/developer-content/blob/main/content/guides/dapps/journal.md#writing-a-solana-program-with-anchor)

## Section Two: Implementing an Escrow for Payment Protection
