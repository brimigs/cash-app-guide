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

For an introduction to solana program development with the anchor framework, review this guide:

- [Basic CRUD dApp on Solana](https://github.com/solana-foundation/developer-content/blob/main/content/guides/dapps/journal.md#writing-a-solana-program-with-anchor)

For an introduction to solana mobile development, take a look at the solana mobile docs:

- [Solana Mobile Introduction](https://docs.solanamobile.com/getting-started/intro)

## Project design overview

Let's start by quickly mapping out the entire dApp design. To create a clone of cash app, we want to have the following functionalities:

1. Account Creation
2. Deposit and Withdraw Funds
3. User-to-User Money Transfer
4. QR Code Generation
5. Connect with Friends
6. Activity Tracking
7. Payment Protection

To enable these functionalies, we will do the following:

1. Write a solana program that allows for users to initialize a new account on-chain and set up a user name _(similar to $Cashtag)_ with [Solana Name Service](https://sns.guide/). With the username being set via SNS, you can then get publickey information direclty from an account's username.
2. Add instructions to the solana program for a user to be able to deposit funds from their wallet into their cash account and withdrawal funds from their cash account into their wallet.
3. Add instructions for a user to be able to directly send funds from their own cash account to another cash account, request funds from a specified cash account, and accept or decline payment requests.
4. Integrate [Solana Pay](https://docs.solanapay.com/) to enable QR code generation. Solana pay also allows you to specify the amount and memo for the requested transaction directly in the QR code.
5. Add an instruction for a user to be able to add friends by pushing the user provided public key to a freinds vector saved to the user's account state, which can then be displayed on the front end similar to cash app.
6. Add an activity tab that queries the cash account state of the connected user to show pending requests and pending payments.
7. Create an escrow account within the program to be able to hold funds for a specifed period of time when requested by the user to enable payment protection. These payments will show as "pending" while they sit in the escrow allowing time for a user to revoke the payment if needed.

## Solana Mobile App Template Set Up

Since this project will be a mobile app, we can get started with the solana mobile expo app template:

```shell
yarn create expo-app --template @solana-mobile/solana-mobile-expo-template
```

Name the project `cash-app-clone` then navigate into the directory.

Follow the [Running the app](https://docs.solanamobile.com/react-native/expo#running-the-app) guide to launch the template as a custom development build and get it running on your andriod emmulator. Once you have built the program and are running a dev client with expo, the emmulator will automatically update everytime you save your code.

Reminder: You must have [fake wallet](https://github.com/solana-mobile/mobile-wallet-adapter/tree/main/android/fakewallet) running on the same android emulator to be able to test out transactions, as explained in the [solana mobile development set up docs](https://docs.solanamobile.com/getting-started/development-setup).

## Writing a Solana Program for Cash App Functionalities

We'll break up the solana program code into a few sections. To start off, lets just enable account creation, deposits, withdrawals, and direct transfers of funds to create a very basic version of cash app.

Ensure you have the [anchor CLI](https://www.anchor-lang.com/docs/cli) installed, then create an anchor directory within `cash-app-clone`. Navigate into the directory and run `anchor init` to initalize a project workspace for the anchor solana program. Now create a `lib.rs` file and we'll get started with the program code.

### Define Your Anchor Program

```rust
use anchor_lang::prelude::*;

declare_id!("11111111111111111111111111111111");

#[program]
pub mod cash_app {
    use super::*;
}
```

### Define Your Account State

```rust
#[account]
#[derive(InitSpace)]
pub struct CashAccount {
    pub owner: Pubkey,
    pub friends: Vec<Pubkey>,
}
```

Naturally, it would make sense to store the account balance in the state, however, we are able to directly query the balance of PDA accounts. So saving the account balance would just add unnecesasary calculations in future instructions.

### Add Instructions

Now that the state is defined, we need to create an instruction to initalize an account when a new user signs up for cash app. This will initialize a new account and save the user's public key into the PDA of the user's cash account.

```rust
#[program]
pub mod cash_app {
    use super::*;

    pub fn initialize_account(ctx: Context<InitializeAccount>) -> Result<()> {
        let cash_account = &mut ctx.accounts.cash_account;
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

The `deposit_funds` function constructs a system instruction to transfer SOL from the user's wallet to the user's cash account PDA. The transfer instruction is executed using `invoke`, which safely performs the cross-program invocation.

Next we need to add an instruction to this program that allows a user to withdraw funds from their cash account:

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

The `withdraw_funds` instruction directly adjusts the lamports _(smallest unit of SOL)_ in the user's cash_account and the user's wallet. Since the cash_account is owned by the program, `try_borrow_mut_lamports` can be used here. A Solana Program can transfer lamports from one account to another without 'invoking' the System program. The fundamental rule is that your program can transfer lamports from any account owned by your program to any account at all. The recipient account does not have to be an account owned by your program. Since lamports can not be created or destroyed when changing account balances, any decrement performed needs to be balanced with an equal increment somewhere else, otherwise you will get an error. In the above `withdraw_funds` instruction, the program is transfering the exact same amount of lamports from the cash account into the users wallet.

Now lets create an instruction for transfering funds from one user to another.

```rust
#[program]
pub mod cash_app {
    use super::*;

    ...

    pub fn transfer_funds(
        ctx: Context<TransferFunds>,
        _recipient: Pubkey,
        amount: u64,
    ) -> Result<()> {
        require!(amount > 0, ErrorCode::InvalidAmount);

        let from_cash_account = ctx.accounts.from_cash_account.to_account_info();
        let to_cash_account = ctx.accounts.to_cash_account.to_account_info();

        **from_cash_account.try_borrow_mut_lamports()? -= amount;
        **to_cash_account.try_borrow_mut_lamports()? += amount;

        Ok(())
    }
}

#[derive(Accounts)]
#[instruction(recipient: Pubkey)]
pub struct TransferFunds<'info> {
    #[account(mut, seeds = [user.key().as_ref()], bump)]
    pub from_cash_account: Account<'info, CashAccount>,
    #[account(mut, seeds = [recipient.key().as_ref()], bump)]
    pub to_cash_account: Account<'info, CashAccount>,
    pub system_program: Program<'info, System>,
    pub user: Signer<'info>,
}
```

In the above instruction, we are once again directly transferring lamports between accounts. The difference here is that the Context data structure `TransferFunds` consists of an additonal account.

Since the seeds for the cash account PDAs are created from the public key of the cash account owner, the instruction needs to take the recipient's publickey as a parameter and pass that to the `TransferFunds` Context data structure. Then the cash_account pda can be derived for both the `from_cash_account` and the `to_cash_account`.

Because both of the accounts are listed in the `#[derive(Accounts)]` macro, they are deserialized and validated so you can simply call both of the accounts with the Context `ctx` to get the account info and them update the account balances from there.

To be able to send funds to another user, similar to Cash App, both users must have created an account. This is because we are sending funds to the user's `cash_account` PDA, not the user's wallet. So each user needs to initialze a cash account by calling the `initialize_account` instruction to create their unique PDA derived from their wallet publickey. We'll need to keep this in mind when designing the UI/UX of the onboarding process for this dApp later on to ensure every user calls the `initialize_account` instruction when signing up for an account.

Now the basic payment functionality is enabled, we want to be able to interact with friends. So we need to add instructions for adding friends, requesting payments from friends, and accepting/rejecting payment requests.

Adding a friend is as simple as just pushing a new publickey to the `friends` vector in the `CashAccount` state.

```rust
#[program]
pub mod cash_app {
    use super::*;

    ...
    pub fn add_friend(ctx: Context<AddFriend>, pubkey: Pubkey) -> Result<()> {
        let cash_account = &mut ctx.accounts.cash_account;
        cash_account.friends.push(pubkey);
        Ok(())
    }
}

#[derive(Accounts)]
pub struct AddFriend<'info> {
    #[account(
        mut,
        seeds = [user.key().as_ref()],
        bump,
    )]
    pub cash_account: Account<'info, CashAccount>,
    #[account(mut)]
    /// CHECK: This account is only used to transfer SOL, not for data storage.
    pub user: AccountInfo<'info>,
    pub system_program: Program<'info, System>,
}
```

### Integrating Multiple Accounts Types

There are several different ways to approach requesting payments from friends. In this example, we will make each payment request its own PDA account in order to simplify querying active requests, deleting completed requests, and updating both the sender and recipent cash accounts. Since this goes beyond the basic solana program set up, we'll implement this later on in the tutorial.

Each time a new payment request is created, the instruction will create a new PDA account that holds data for the payment's sender, recipient, and amount.

To have multiple account types within one program, you just need to define the data structure for each account type and have an seperate instructions to be able to initlize each account type. We already have the state data structure and init account instruction for the cash account, now we'll just add this for the pending request account.

```rust
#[account]
#[derive(InitSpace)]
pub struct PendingRequest {
    pub sender: Pubkey,
    pub recipient: Pubkey,
    pub amount: u64,
}

#[derive(Accounts)]
pub struct InitializeAccount<'info> {
    #[account(
        init,
        seeds = [b"pending-request", user.key().as_ref()],
        bump,
        payer = user,
        space = 8 + PendingRequest::INIT_SPACE
    )]
    pub pending_request: Account<'info, PendingRequest>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[program]
pub mod cash_app {
    use super::*;

    ...

    pub fn new_pending_request(ctx: Context<InitializeAccount>, recipient: Pubkey, amount: u64) -> Result<()> {
        let pending_request = &mut ctx.accounts.pending_request;
        pending_request.sender = *ctx.accounts.user.key;
        pending_request.recipient = recipient;
        pending_request.amount = amount;
        Ok(())
    }
}
```

Now that we are able to send payment requests, we need to be able to accept or decline those payments. So let's add in those instructions now.

```rust
#[program]
pub mod cash_app {
    use super::*;

    ...

    pub fn accept_request(ctx: Context<ProcessRequest>) -> Result<()> {
        let recipient = ctx.accounts.pending_request.recipient;
        let amount = ctx.accounts.pending_request.amount;

        transfer_funds(ctx, recipient, amount)

        Ok(())
    }

    pub fn decline_request(_ctx: Context<ProcessRequest>) -> Result<()> {
        Ok(())
    }
}

#[derive(Accounts)]
pub struct ProcessRequest<'info> {
    #[account(
        mut,
        seeds = [b"pending-request", user.key().as_ref()],
        bump,
        close = user,
    )]
    pub pending_request: Account<'info, PendingRequest>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```

In the above code, all we need to do to decline a request is to close the account on chain, by specifying the `close` constraint in the account macro for the `DeclineRequest` data structure, the account simply closes when the correct signer signs the `decline_request` instruction.

For `pending_requests`, we also want the account to close upon completion of the instruction but the requested funds need to be transfered first. We can just get the needed information from the account state and call the `trasnsfer_funds` function we defined earlier.

We're now able to deposit funds, withdraw funds, send funds to another user, request funds from another user, add friends, and accept/decline requests, which covers all of the functionaltiy in cash app.

### Additional Solana Program Development Information

If there is any confusion on the above anchor macros, structs, or functions defined, please refer to the [Basic CRUD dApp on Solana Guide](https://github.com/solana-foundation/developer-content/blob/main/content/guides/dapps/journal.md#writing-a-solana-program-with-anchor) for a guide with a more granular explaination.

Creating tests is out of scope for this guide, however, it is important to prioritize testing when developing a solana program. For information on testing with anchor, read these docs:

// FIXME: Do we have solana docs on anchor testing?? I thought we did but cant find them.

For more in-depth understanding of the anchor framework, review [The Anchor Book](https://book.anchor-lang.com/).

## Connecting a Solana Program to a React-Native Expo App

Now that we have a working solana program, we need to integrate this with the UI of the dApp.

### Android Emmulator

Lets get the android emmulator running so we can see in real time the UI updates that we will make throughout this guide.

You must have an EAS account and be logged into your account in the EAS cli, to set this up follow [the expo documentation](https://docs.expo.dev/build/setup/).

Navigate to the `cash-app-clone` directory in your terminal and run:

```shell
eas build --profile development --platform android
```

Then in a new terminal window run:

```shell
npx expo start --dev-client
```

Install the build on your android emmulator and keep it running in a seperate window. Everytime you save a file, the emmulator will refresh.

### Initial Program Connection

First, we need to deploy the anchor program. For testing purposes, you can either deploy to your localnet or to devnet. Devnet is beneficial when you wish to share with others, since everyone has access the the devnet rpc endpoint. On the other hand, localnet enables for faster iteration, unlimited airdrops, and network customizatioln. However, localnet is ran locally on your computer with `solana-test-validator` so your program ID will not be compatible with anyone who is not using your computer's localhost. Since I want this dApp to be working on devnet for anyone to be able to try out, I'll be using devnet throughout this guide.

1. Run `anchor build` to build your program's workspace. This targets Solana's BPF runtime and emits each program's IDL in the `target/idl` directory.
2. Run `anchor deploy --provider.cluster devnet` to deploy your program in the workspace to the specified cluster and generate a program ID. If you do choose to deploy to localnet, you must be running `solana-test-validator` to be able to deploy.
3. Run `anchor keys sync` to sync the program's `declare_id!` pubkey with the program's actual pubkey

Now that we have the program ID and program's IDL, we can start to connect to the front end.

We can create a custom hook that accepts the public key of the user as a parameter that is designed to interact with a solana program. By providing the program ID, the rpc endpoint that the program was deployed to, the IDL of the program, and the PDA of a specified user, we can create the logic required to manager interaction with the solana program on the specified network. Create a new file under `utils/useCashAppProgram.tsx`, to implement this function.

```typescript
export function UseCashAppProgram(user: PublicKey) {
  const cashAppProgramId = new PublicKey(
    "BxCbQks4iaRvfCnUzf3utYYG9V53TDwVLxA6GGBnhci4"
  );

  const [connection] = useState(
    () => new Connection("https://api.devnet.solana.com")
  );

  const [cashAppPDA] = useMemo(() => {
    const counterSeed = user.toBuffer();
    return PublicKey.findProgramAddressSync([counterSeed], cashAppProgramId);
  }, [cashAppProgramId]);

  const cashAppProgram = useMemo(() => {
    return new Program<CashAppProgram>(
      idl as CashAppProgram,
      cashAppProgramId,
      { connection }
    );
  }, [cashAppProgramId]);

  const value = useMemo(
    () => ({
      cashAppProgram: cashAppProgram,
      cashAppProgramId: cashAppProgramId,
      cashAppPDA: cashAppPDA,
    }),
    [cashAppProgram, cashAppProgramId, cashAppPDA]
  );

  return value;
}
```

Since this funciton takes in the public key of the connected wallet and we designed the cash app pda to be generated based on the user's public key, we can easily calculate what the public key of the cash app PDA for each individual user is.

Since the IDL is generated as a JSON file when building the program, we can just import it to this file.

This funciton returns:

- `cashAppPDA` - The connect user's Program Derived Address (PDA) for their cash account
- `cashAppProgramID` - The public key of the deployed solana program on devnet
- `cashAppProgram` - The cash app program which provides the IDL deserialized client representation of an Anchor program.

The `Program` class is an import from `@coral-xyz/anchor`. This API is a one stop shop for all things related to communicating with on-chain programs. It enables sending transactions, deserializing accounts, decoding instruction data, listening to events, etc.

The `Program` object provides `namespaces`, which map one-to-one to program methods and accounts, which we will be using a lot later in this project. The `namespace` is generally used as follows: `program.<namespace>.<program-specifc-method>`

### Styling and Themes

React Native uses a styling system that is based on the standard CSS properties but adapted for mobile development. Styles are written in JavaScript using objects, which allows the benefit of leveraging JavaScript's power to dynamically generate styles. In order to mimic the look and feel of cash app, we'll create a StyleSheet Object that we can use throughout this dApp. This will create a monochrome greyscale color pallete with bold text and rounded shapes.

```jsx
import { StyleSheet, Dimensions } from "react-native";

const { width } = Dimensions.get("window");

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: "#141414",
    alignItems: "center",
    justifyContent: "center",
  },
  header: {
    backgroundColor: "#1b1b1b",
    width: "100%",
    padding: 20,
    alignItems: "center",
  },
  headerText: {
    color: "#fff",
    fontSize: 24,
    fontWeight: "bold",
  },
  button: {
    width: 80,
    height: 80,
    justifyContent: "center",
    alignItems: "center",
    backgroundColor: "#333", // Darker button background
    borderRadius: 40,
  },
  buttonGroup: {
    flexDirection: "column",
    paddingVertical: 4,
  },
  buttonText: {
    color: "#fff",
    fontSize: 18,
    fontWeight: "600",
    textAlign: "center",
  },
  cardContainer: {
    width: width - 40,
    backgroundColor: "#222",
    borderRadius: 20,
    padding: 20,
    marginVertical: 10,
    shadowColor: "#000",
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.22,
    shadowRadius: 2.22,
    elevation: 3,
  },
  modalView: {
    backgroundColor: "#444",
    padding: 35,
    alignItems: "center",
    borderTopLeftRadius: 50,
    borderTopRightRadius: 50,
    shadowColor: "#000",
    shadowOffset: {
      width: 0,
      height: -2, // Negative value to lift the shadow up
    },
    shadowOpacity: 0.8,
    shadowRadius: 4,
    elevation: 5,
    width: "100%", // Ensure the modal occupies full width
    height: "40%", // Only take up half the screen height
  },
  cardTitle: {
    fontSize: 20,
    fontWeight: "bold",
    marginBottom: 5,
  },
  cardContent: {
    fontSize: 16,
    color: "#666",
  },
});

export default styles;
```

Along with setting the `StyleSheet`, we also need to update the theme of the dApp. A theme creates a more uniform look and feel throughout the entire application. Navigate to `App.tsx`, and update the code to only use `DarkTheme`. You should now see the template update to a dark background with light text and a dark navigation bar with light icons.

### Navigation Bar and Pages Set up

To follow the UI/UX of cash app, we'll need the following screens: Home, Pay, Scan, and Activity.

Navigate to `HomeNavigator.tsx` and update the `<Tab.Navigator>` to include the following screens:

```typescript
<PaperProvider theme={theme}>
  <Tab.Navigator
    screenOptions={({ route }) => ({
      header: () => <TopBar />,
      tabBarIcon: ({ focused, color, size }) => {
        switch (route.name) {
          case "Home":
            return (
              <MaterialCommunityIcon
                name={focused ? "home" : "home-outline"}
                size={size}
                color={color}
              />
            );
          case "Pay":
            return (
              <MaterialCommunityIcon
                name={focused ? "currency-usd" : "currency-usd"}
                size={size}
                color={color}
              />
            );
          case "Scan":
            return (
              <MaterialCommunityIcon
                name={focused ? "qrcode-scan" : "qrcode-scan"}
                size={size}
                color={color}
              />
            );
          case "Activity":
            return (
              <MaterialCommunityIcon
                name={focused ? "clock-outline" : "clock-outline"}
                size={size}
                color={color}
              />
            );
        }
      },
    })}
  >
    <Tab.Screen name="Home" component={HomeScreen} />
    <Tab.Screen name="Pay" component={PayScreen} />
    <Tab.Screen name="Scan" component={ScanScreen} />
    <Tab.Screen name="Activity" component={ActivityScreen} />
  </Tab.Navigator>
</PaperProvider>
```

In addition to this, you'll need to create new files for each of these screens. Navigate to
`src/screens` and create a file for `PayScreen.tsx`, `ScanScreen.tsx`, and `ActivityScreen.tsx`.

Each file needs to have a function correlating to the screen name that follows the same format as the HomeScreen in this template.

```typescript
export function HomeScreen() {
  return <View style={styles.screenContainer}></View>;
}
```

### Creating Components

Throughout this project, we'll be using a modular approach to building features, so we can focus on one component at a time.

Let's start with the home screen. To mimic cash app, all we need is a container that displays your account balance, a button to deposit funds into your account, and a button to withdraw funds from your account.

In the expo template we are using, there is already similar funcitonality. However, this code is for your connected wallet balance rather than the cash account's balance. So we need to connect this feature to our deployed solana program and query that balance instead.

First simplify the home screen to just:

```typescript
export function HomeScreen() {
  const { selectedAccount } = useAuthorization();

  return (
    <View style={styles.screenContainer}>
      {selectedAccount ? (
        <>
          <AccountDetailFeature />
        </>
      ) : (
        <>
          <Text style={styles.headerTextLarge}>Solana Cash App</Text>
          <Text style={styles.text}>
            {" "}
            Sign in with Solana (SIWS) to link your wallet.
          </Text>
          <SignInFeature />
        </>
      )}
    </View>
  );
}
```

Then click into `AccountDetailFeature` and update the styling to use `cardContainer`, add in a "Cash Balance" label for the card container, and delete teh `AccountTokens` component. as shown below:

```typescript
export function AccountDetailFeature() {
  const { selectedAccount } = useAuthorization();

  if (!selectedAccount) {
    return null;
  }
  const theme = useTheme();

  return (
    <>
      <View style={styles.cardContainer}>
        <Text variant="titleMedium" style={styles.headerText}>
          Cash Balance
        </Text>
        <View style={{ alignItems: "center" }}>
          <AccountBalance address={selectedAccount.publicKey} />
          <AccountButtonGroup address={selectedAccount.publicKey} />
        </View>
      </View>
    </>
  );
}
```

NOTE: The `StyleSheet` that we created earlier should be imported to every page.

Now click into the `AccountBalance` function. We need to change this to query the cash account rather than the user's connected wallet. All that needs to be changed is the public key that is being passed through the `useGetBalance` function. We can grab the `cashAppPDA` from the `UseCashAppProgram` function we created earlier.

```typescript
export function AccountBalance({ address }: { address: PublicKey }) {
  const { cashAppPDA } = UseCashAppProgram(address);

  const query = useGetBalance(cashAppPDA);
  const theme = {
    ...MD3DarkTheme,
    ...DarkTheme,
    colors: {
      ...MD3DarkTheme.colors,
      ...DarkTheme.colors,
    },
  };

  return (
    <>
      <View style={styles.accountBalance}>
        <Text variant="displayMedium" theme={theme}>
          ${query.data ? lamportsToSol(query.data) : "0.00"}
        </Text>
      </View>
    </>
  );
}
```

### Using Namespaces

Next, we need to update the buttons to deposit and withdraw funds. Go to the `AccountButtonGroup` function.

To be able to call and execute an instruction from the deployed solana program, we can use the program namespaces which map one-to-one to program methods and accounts.

```typescript
const [connection] = useState(
  () => new Connection("https://api.devnet.solana.com")
);

const depositFunds = useCallback(
  async (program: Program<CashApp>) => {
    let signedTransactions = await transact(
      async (wallet: Web3MobileWallet) => {
        const [authorizationResult, latestBlockhash] = await Promise.all([
          authorizeSession(wallet),
          connection.getLatestBlockhash(),
        ]);

        const depositInstruction = await program.methods
          .depositFunds(pubkey, newDepositAmount)
          .accounts({
            user: authorizationResult.publicKey,
            fromCashAccount: cashAppPDA,
          })
          .instruction();

        const depositTransaction = new Transaction({
          ...latestBlockhash,
          feePayer: authorizationResult.publicKey,
        }).add(depositInstruction);

        const signedTransactions = await wallet.signTransactions({
          transactions: [depositTransaction],
        });

        return signedTransactions[0];
      }
    );

    let txSignature = await connection.sendRawTransaction(
      signedTransactions.serialize(),
      {
        skipPreflight: true,
      }
    );

    const confirmationResult = await connection.confirmTransaction(
      txSignature,
      "confirmed"
    );

    if (confirmationResult.value.err) {
      throw new Error(JSON.stringify(confirmationResult.value.err));
    } else {
      console.log("Transaction successfully submitted!");
    }
  },
  [authorizeSession, connection, cashAppPDA]
);
```

This funciton uses React's useCallback hook to create a memoized callback function that handles the process of depositing funds within the connected solana program. It accepts a `Program` parameter which is an Anchor program interface for the `CashApp` dApp.

Since the `namespace` is generally used as follows: `program.<namespace>.<program-specifc-method>`, in the above code, we are creating an `instruction` to `depositFunds` with the specified `accounts`.

Then this instruction can be added to a `Transaction` and signed with the connected wallet.

Lastly, the signed transaction is then sent by using the `sendRawTransaction` method from `connection` object.

The `connection` object is an instance of the `Connection` class from the `solanaweb3.js` library, which is a connection to a fullnode JSON RPC endpoint.

Now that we have the function for `depositFunds`, you'll need to do follow the same formate to create a `withdrawFunds` funciton using the program namespace for the withdrawFunds instruction.

```typescript
const withdrawInstruction = await program.methods
  .withdrawFunds(pubkey, newDepositAmount)
  .accounts({
    user: authorizationResult.publicKey,
    fromCashAccount: cashAppPDA,
  })
  .instruction();
```

**Additional documentation:**

- [Transactions and Instructions](https://solana.com/docs/core/transactions)
- [Connection Class](https://solana-labs.github.io/solana-web3.js/classes/Connection.html)
- Library for [wallets](https://github.com/solana-mobile/mobile-wallet-adapter/tree/main/android/walletlib) to provide the Mobile Wallet Adapter transaction signing services to dapps

Npm packages to be installed and imported:

- @solana-mobile/mobile-wallet-adapter-protocol-web3js
- @coral-xyz/anchor
- @solana/web3.js

Now we can connect these functions to buttons on the UI. We'll follow a very similar structure to the current `AccountButtonGroup` function, but we need different functionality. So delete everything within the funciton. Since cash app also uses modals when clicking on the "Add Cash" and "Cash Out" buttons, we'll have a withdraw and deposit modal. We'll also need to take in a user input value for the amount to be deposited or withdrawn. We'll also need the `depositFunds` and `withdrawFunds` functions we just created.

```typescript
export function AccountButtonGroup({ address }: { address: PublicKey }) {
  const [showWithdrawModal, setShowWithdrawModal] = useState(false);
  const [showDepositModal, setShowDepositModal] = useState(false);
  const [genInProgress, setGenInProgress] = useState(false);
  const [depositAmount, setDepositAmount] = useState(new anchor.BN(0));
  const newDepositAmount = new anchor.BN(depositAmount * 1000000000);
  const [withdrawAmount, setWithdrawAmount] = useState(new anchor.BN(0));
  const newWithdrawAmount = new anchor.BN(withdrawAmount * 1000000000);
  const { authorizeSession, selectedAccount } = useAuthorization();
  const { cashAppProgram } = UseCashAppProgram(address);

  const [connection] = useState(
    () => new Connection("https://api.devnet.solana.com")
  );

  const DepositModal = () => (
    <Modal
      animationType="slide"
      transparent={true}
      visible={showDepositModal}
      onRequestClose={() => {
        setShowDepositModal(!showDepositModal);
      }}
    >
      <View style={styles.bottomView}>
        <View style={styles.modalView}>
          <Text style={styles.buttonText}>Add Cash</Text>
          <TextInput
            label="Amount"
            value={depositAmount}
            onChangeText={setDepositAmount}
            keyboardType="numeric"
            mode="outlined"
            style={{
              marginBottom: 10,
              backgroundColor: "#ccc",
              width: "80%",
              marginTop: 10,
            }}
          />
          <Button
            mode="contained"
            style={styles.modalButton}
            onPress={async () => {
              setDepositModalVisible(!showDepositModal);
              if (genInProgress) {
                return;
              }
              setGenInProgress(true);
              try {
                if (!cashAppProgram || !selectedAccount) {
                  console.warn(
                    "Program/wallet is not initialized yet. Try connecting a wallet first."
                  );
                  return;
                }
                const deposit = await depositFunds(cashAppProgram);

                alertAndLog(
                  "Funds deposited into cash account ",
                  "See console for logged transaction."
                );
                console.log(deposit);
              } finally {
                setGenInProgress(false);
              }
            }}
          >
            Add
          </Button>
          <TouchableOpacity
            style={{ position: "absolute", bottom: 25 }}
            onPress={() => setDepositModalVisible(false)}
          >
            <Button>Close</Button>
          </TouchableOpacity>
        </View>
      </View>
    </Modal>
  );

  const WithdrawModal = () => (
    <Modal
      animationType="slide"
      transparent={true}
      visible={showWithdrawModal}
      onRequestClose={() => {
        setShowWithdrawModal(!showWithdrawModal);
      }}
    >
      <View style={styles.bottomView}>
        <View style={styles.modalView}>
          <Text style={styles.buttonText}>Cash Out</Text>
          <TextInput
            label="Amount"
            value={withdrawAmount}
            onChangeText={setWithdrawAmount}
            keyboardType="numeric"
            mode="outlined"
            style={{
              marginBottom: 20,
              backgroundColor: "#ccc",
              width: "80%",
              marginTop: 50,
            }}
          />
          <Button
            mode="contained"
            style={styles.modalButton}
            onPress={async () => {
              setShowWithdrawModal(!withdrawModalVisible);
              if (genInProgress) {
                return;
              }
              setGenInProgress(true);
              try {
                if (!cashAppProgram || !selectedAccount) {
                  console.warn(
                    "Program/wallet is not initialized yet. Try connecting a wallet first."
                  );
                  return;
                }
                const deposit = await withdrawFunds(cashAppProgram);

                alertAndLog(
                  "Funds withdrawn from cash account ",
                  "See console for logged transaction."
                );
                console.log(deposit);
              } finally {
                setGenInProgress(false);
              }
            }}
          >
            Withdraw
          </Button>
          <TouchableOpacity
            style={{ position: "absolute", bottom: 25 }}
            onPress={() => setShowWithdrawModal(false)}
          >
            <Button>Close</Button>
          </TouchableOpacity>
        </View>
      </View>
    </Modal>
  );
  return (
    <>
      <View style={styles.buttonRow}>
        <DepositModal />
        <WithdrawModal />
      </View>
    </>
  );
}
```

That wraps up all the functionality we need on the home screen for a cash app clone. Now we can move onto the pay screen, which involves transfering funds from one user to another.

Now lets create the components needed for the pay screen. In cash app, the pay screen is simply a key pad with `request` and `pay` buttons that redirect you to another screen.

So the pay screen is mainly some UI work. We need to be able to type in a numerical value via a keyboard, handle the input value, select currency via a small modal, and navigate to the request and pay pages via buttons. Here is the code below:

```typescript
type HomeScreenNavigationProp = NativeStackNavigationProp<
  RootStackParamList,
  "Home"
>;

type Props = {
  navigation: HomeScreenNavigationProp;
};

const App: React.FC<Props> = ({ navigation }) => {
  const [inputValue, setInputValue] = useState("");
  const [modalVisible, setModalVisible] = useState(false);

  // Function to handle input from keypad
  const handleInput = (value: string) => {
    setInputValue(inputValue + value);
  };

  // Function to handle backspace
  const handleBackspace = () => {
    setInputValue(inputValue.slice(0, -1));
  };

  type NumberButtonProps = {
    number: string;
  };

  // Create a single button for the keypad
  const NumberButton: React.FC<NumberButtonProps> = ({ number }) => (
    <TouchableOpacity style={styles.button} onPress={() => handleInput(number)}>
      <Text style={styles.buttonText}>{number}</Text>
    </TouchableOpacity>
  );

  const CurrencySelectorModal = () => (
    <Modal
      animationType="slide"
      transparent={true}
      visible={modalVisible}
      onRequestClose={() => {
        setModalVisible(!modalVisible);
      }}
    >
      <View style={styles.bottomView}>
        <View style={styles.modalView}>
          <Text style={styles.buttonText}>Select Currency</Text>
          <View style={styles.centeredView}>
            <TouchableOpacity
              style={styles.fullWidthButton}
              onPress={() => setModalVisible(false)}
            >
              <Text style={styles.currencyText}>
                {" "}
                <MaterialCommunityIcon
                  name="currency-usd"
                  size={30}
                  color="white"
                />
                US Dollars
              </Text>
            </TouchableOpacity>
            <TouchableOpacity
              style={styles.fullWidthButton}
              onPress={() => setModalVisible(false)}
            >
              <Text style={styles.currencyText}>
                {" "}
                <MaterialCommunityIcon name="bitcoin" size={30} color="white" />
                Bitcoin
              </Text>
            </TouchableOpacity>
          </View>
          <TouchableOpacity
            style={{ position: "absolute", bottom: 25 }}
            onPress={() => setModalVisible(false)}
          >
            <Text style={styles.mediumButtonText}>Close</Text>
          </TouchableOpacity>
        </View>
      </View>
    </Modal>
  );

  return (
    <View style={styles.container}>
      <CurrencySelectorModal />
      <View style={styles.displayContainer}>
        <Text style={styles.displayText}>${inputValue || "0"}</Text>
        <TouchableOpacity
          style={{ position: "relative", marginTop: 15 }}
          onPress={() => setModalVisible(true)}
        >
          <Text style={styles.smallButtonText}>
            USD{" "}
            <MaterialCommunityIcon
              name="chevron-down"
              size={15}
              color="white"
            />
          </Text>
        </TouchableOpacity>
      </View>
      <View style={styles.keypad}>
        <View style={styles.row}>
          {[1, 2, 3].map((number) => (
            <NumberButton key={number} number={number.toString()} />
          ))}
        </View>
        <View style={styles.row}>
          {[4, 5, 6].map((number) => (
            <NumberButton key={number} number={number.toString()} />
          ))}
        </View>
        <View style={styles.row}>
          {[7, 8, 9].map((number) => (
            <NumberButton key={number} number={number.toString()} />
          ))}
        </View>
        <View style={styles.row}>
          <NumberButton number="." />
          <NumberButton number="0" />
          <TouchableOpacity style={styles.button} onPress={handleBackspace}>
            <Text style={styles.buttonText}>⌫</Text>
          </TouchableOpacity>
        </View>
      </View>
      <View style={styles.buttonRow}>
        <Button
          mode="contained"
          style={styles.sideButton}
          onPress={() => navigation.navigate("Receive", { inputValue })}
        >
          Request
        </Button>
        <Button
          mode="contained"
          style={styles.sideButton}
          onPress={() => navigation.navigate("Send", { inputValue })}
        >
          Pay
        </Button>
      </View>
    </View>
  );
};
```

Now the request and pay pages is where the real logic comes in and the program interaction.

For the pay page, we'll need to implement the `transferFunds` function from the cash app solana program. To do this, we'll be using the same process that was described for `depositFunds`. However, the `TransferFunds` struct described in the CashApp Solana Program requires 2 accounts rather than the one account that is required for `depositFunds`. So what needs to change is simply to add calculations of the PDAs of both the sender account and the recipient's account, as shown below:

```typescript
const [recipientPDA] = useMemo(() => {
  const counterSeed = recipient.toBuffer();
  return PublicKey.findProgramAddressSync([counterSeed], cashAppProgramId);
}, [cashAppProgramId]);

const transferInstruction = await program.methods
  .transferFunds(pubkey, newTransferAmount)
  .accounts({
    user: authorizationResult.publicKey,
    fromCashAccount: cashAppPDA,
    toCashAccount: recipientPDA,
  })
  .instruction();
```

In order to calculate the recipient's PDA, the public key of the recipient must be passed through as a parameter of the `transferFunds` function, along with the amount to transfer and the public key of the signer.

## Enabling QR Code functionality with Solana Pay

To mimic the QR code funcitonality in Cash App, you can simply use the `@solana/pay` JavaScript SDK. For more information, refer to the [Solana Pay API Reference](https://docs.solanapay.com/api/core).

The `encodeURL` function takes in an amount and a memo to encode a Solana Pay URL for a specifc transaction.

Typically, this function is paired with `createQR` to generate a QR code with the Solana Pay URL. As of today, Solana Pay's current version of the `createQR` funciton is not compatible with react-native, so we will need to use a different QR code generator that is react-native compatible. In this guide, we'll input the url into `QRCode` from `react-native-qrcode-svg`. It does not have the same QR code styling as the Solana Pay `createQR`, but it still correctly generates the needed QR code.

For simplicity, this functionality will live on its own screen, which we already defined earlier as the Scan Screen. Similarly to the home screen, navigate to `ScanScreen.tsx` and set up the following function:

```typescript
export function ScanScreen() {
  const { selectedAccount } = useAuthorization();

  return (
    <View style={styles.container}>
      {selectedAccount ? (
        <View style={styles.container}>
          <SolanaPayButton address={selectedAccount.publicKey} />
        </View>
      ) : (
        <>
          <Text style={styles.headerTextLarge}>Solana Cash App</Text>
          <Section description="Sign in with Solana (SIWS) to link your wallet." />
          <SignInFeature />
        </>
      )}
    </View>
  );
}
```

Now we need to create the `SolanaPayButton` component. Create a file under `src/components/solana-pay/solana-pay-ui.tsx`. In cash app, the QR code is just a link to the users cash app profile and is a static image in the app. However, the solana pay QR code is actually uniquely generated for each requested transaction, so the QR displayed includes the amount, memo, and the recipient's publickey information. So our UI/UX will function slightly different than cash app in this section.

To still follow the look and feel of cash app, we'll allow most of the screen to display the QR code and have a button at the bottom for a modal that has amount and memo input fields and a generate QR code button. On clicking the "Create QR" button, we'll want to generate a new Solana Pay URL and send that value outside of the modal to the Scan Screen so that the screen will render and display the new QR code.

We can do this with the solana pay api, state handling, conditional rendering, and data submission between the two components, as shown below:

```typescript
export function SolanaPayButton({ address }: { address: PublicKey }) {
  const [showPayModal, setShowPayModal] = useState(false);

  const [url, setUrl] = useState("");

  return (
    <>
      <View>
        <View
          style={{
            height: 200,
            width: 200,
            justifyContent: "center",
            alignItems: "center",
            alignSelf: "center",
            marginBottom: 200,
            marginTop: 200,
          }}
        >
          {url ? (
            <>
              <View
                style={{
                  height: 350,
                  width: 350,
                  justifyContent: "center",
                  alignItems: "center",
                  alignSelf: "center",
                  backgroundColor: "#333",
                  borderRadius: 25,
                }}
              >
                <QRCode
                  value={url}
                  size={300}
                  color="black"
                  backgroundColor="white"
                />
              </View>
            </>
          ) : (
            <View
              style={{
                height: 350,
                width: 350,
                justifyContent: "center",
                alignItems: "center",
                borderWidth: 1,
                borderColor: "#ccc",
                backgroundColor: "#333",
                borderRadius: 25,
              }}
            >
              <Text style={styles.text2}> Generate a QR Code to display. </Text>
            </View>
          )}
          <Text style={styles.text}> Scan to Pay </Text>
          <Text style={styles.text3}> $BRIMIGS </Text>
        </View>
        <SolPayModal
          hide={() => setShowPayModal(false)}
          show={showPayModal}
          address={address}
          setParentUrl={setUrl}
        />
        <Button
          mode="contained"
          onPress={() => setShowPayModal(true)}
          style={styles.button}
        >
          Create New QR Code
        </Button>
      </View>
    </>
  );
}

export function SolPayModal({
  hide,
  show,
  address,
  setParentUrl,
}: {
  hide: () => void;
  show: boolean;
  address: PublicKey;
  setParentUrl: (url: string) => void;
}) {
  const [memo, setMemo] = useState("");
  const [amount, setAmount] = useState("");

  const handleSubmit = () => {
    const number = BigNumber(amount);
    const newUrl = encodeURL({
      recipient: address,
      amount: number,
      memo,
    }).toString();
    setParentUrl(newUrl);
    hide();
  };

  return (
    <AppModal
      title="Pay"
      hide={hide}
      show={show}
      submit={handleSubmit}
      submitLabel="Create QR"
      submitDisabled={!memo || !amount}
    >
      <View style={{ padding: 20 }}>
        <TextInput
          label="Amount"
          value={amount}
          onChangeText={setAmount}
          keyboardType="numeric"
          mode="outlined"
          style={{ marginBottom: 20, backgroundColor: "#f0f0f0" }}
        />
        <TextInput
          label="Memo"
          value={memo}
          onChangeText={setMemo}
          mode="outlined"
          style={{ marginBottom: 5, backgroundColor: "#f0f0f0" }}
        />
      </View>
    </AppModal>
  );
}
```

## Connecting User Names with Public Keys via Solana Name Service

Solana Name Service _(SNS)_ enables a human-readable name to be mapped to a SOL address. By implementing SNS, we can easily prompt a user to create a user name _(which will become their SNS name behind the scenes)_ and that name will directly map to the users wallet address.

Solana Name Service has two functions that we can implement throughout this dapp to simplify a lot of the front end:

- `getDomainKeySync` - a function that returns the public key associated with the provided domain name. This can be implemented anywhere there is a user input for a public key. Now the user only needs to type in a username when searching for an account, exactly as you do with cash app. This is what SNS calls a [direct lookup](https://sns.guide/domain-name/domain-direct-lookup.html).

- `reverseLookup` - an asynchronous function that returns the domain name of the provided public key.This can be implemented anywhere in the UI where you want to display the username. This is what SNS calls a [reverse lookup](https://sns.guide/domain-name/domain-reverse-lookup.html)

To showcase this, lets update the transfers funds function to now accept a user name as a parameter rather than a public key and integrate the SNS API.

```typescript
const transferFunds = useCallback(
  async (program: Program<CashApp>) => {
    let signedTransactions = await transact(
      async (wallet: Web3MobileWallet) => {
        const [authorizationResult, latestBlockhash] = await Promise.all([
          authorizeSession(wallet),
          connection.getLatestBlockhash(),
        ]);

        const { pubkey } = getDomainKeySync(userName);

        const [recipientPDA] = useMemo(() => {
          const counterSeed = pubkey.toBuffer();
          return PublicKey.findProgramAddressSync(
            [counterSeed],
            cashAppProgramId
          );
        }, [cashAppProgramId]);

        const transferInstruction = await program.methods
          .transferFunds(pubkey, newTransferAmount)
          .accounts({
            user: authorizationResult.publicKey,
            fromCashAccount: cashAppPDA,
            toCashAccount: recipientPDA,
          })
          .instruction();

        const transferTransaction = new Transaction({
          ...latestBlockhash,
          feePayer: authorizationResult.publicKey,
        }).add(transferInstruction);

        const signedTransactions = await wallet.signTransactions({
          transactions: [transferTransaction],
        });

        return signedTransactions[0];
      }
    );

    let txSignature = await connection.sendRawTransaction(
      signedTransactions.serialize(),
      {
        skipPreflight: true,
      }
    );

    const confirmationResult = await connection.confirmTransaction(
      txSignature,
      "confirmed"
    );

    if (confirmationResult.value.err) {
      throw new Error(JSON.stringify(confirmationResult.value.err));
    } else {
      console.log("Transaction successfully submitted!");
    }
  },
  [authorizeSession, connection, cashAppPDA]
);
```

This implementation can be integrated everywhere in the application where an input requires a public key, enabling the user experience to be identical to that of a web2 application.
