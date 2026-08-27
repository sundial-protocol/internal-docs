# Sundial Testnet — User Guide

This is an **end-user** guide: how to get a wallet, get testnet sBTC from the
faucet, and (optionally) explore the Sundial demo dashboard. It does not
assume any programming background.

If you're a developer integrating with the node's HTTP API, see
[api.md](./api.md) instead. If you're looking for security/rate-limit
behavior of the faucet endpoint, see
[security.md](./security.md#no-built-in-authentication-or-rate-limiting).

## What is the Sundial testnet?

Sundial is building Bitcoin-native yield infrastructure. The testnet lets you
try the product using **test tokens that have no real value** — nothing you
do here touches real BTC, ADA, or money.

The faucet at
[sundialprotocol.com/testnet/faucet](https://www.sundialprotocol.com/testnet/faucet)
gives out free **testnet sBTC** — Sundial's synthetic BTC token on Midgard,
Sundial's Layer 2. You'll receive it at an address on that L2, which uses the
same address format as Cardano's testnet ("Preprod").

> Everything below is test-only. Never send real funds to a testnet address,
> and don't reuse a wallet that holds real money for testnet activity.

## What you need

Just a **Cardano-compatible wallet browser extension**, set to the testnet
network. You do not need any ADA, BTC, or other funds to get started — the
faucet is how you receive your first testnet sBTC.

Any of these extensions work (all support the CIP-30 standard the faucet page
and Sundial's L2 rely on):

| Wallet | Notes |
| --- | --- |
| [Eternl](https://eternl.io/) | Recommended — easy network switching, widely used |
| [Lace](https://www.lace.io/) | Recommended — official IOG wallet, simple UI |
| [Typhon](https://typhonwallet.io/) | |
| [Yoroi](https://yoroi-wallet.com/) | |
| [GeroWallet](https://gerowallet.io/) | |
| [NuFi](https://nu.fi/) | |

You only need one. Install it as a browser extension, create or restore a
wallet, and keep going.

> A separate Bitcoin wallet (e.g. for the demo dashboard's "Connect Wallet"
> button) is optional and covered in [Explore the demo dashboard](#optional-explore-the-demo-dashboard)
> below — you don't need one just to claim faucet funds.

## Step 1 — Switch your wallet to testnet and copy your address

1. Open your wallet extension.
2. Find its network setting (usually in Settings, sometimes a network name in
   the top bar) and switch it from **Mainnet** to **Testnet** / **Preprod**.
   Every wallet listed above supports this — the exact menu wording varies.
3. Once switched, open the **Receive** screen and copy your **payment
   address**. A testnet address always starts with `addr_test1`.

If your wallet is still on Mainnet, its address will start with `addr1`
instead — that won't work with the faucet (see
[Troubleshooting](#troubleshooting-faucet-errors) below).

## Step 2 — Request testnet sBTC from the faucet

1. Go to
   [sundialprotocol.com/testnet/faucet](https://www.sundialprotocol.com/testnet/faucet).
2. Paste your `addr_test1…` address into the **Sundial testnet address**
   field.
3. Click **Request testnet sBTC**.
4. On success you'll see a confirmation with:
   - **Grant amount** — how much testnet sBTC you received.
   - **Next eligible at** — when this address can claim again (each address
     has a cooldown between claims).
   - **Claim ID** and **Transaction hash** — reference IDs for the claim; the
     transaction hash can be copied with the button next to it.

   ![Faucet success screen](./assets/faucet-success.png)

That's it — no wallet connection or signature is required for this step, you
just paste an address you control.

## Troubleshooting faucet errors

If the request doesn't succeed, the page shows a specific reason:

| What you see | What it means | What to do |
| --- | --- | --- |
| "Enter a valid Sundial testnet payment address" / address must start with `addr_test1` | The field is empty or the address isn't a testnet address | Make sure your wallet is switched to Testnet/Preprod, then copy the address again |
| "This address has already claimed recently" (Cooldown) | You already claimed with this address inside its cooldown window | Wait until the "Next eligible at" time shown, or use a different testnet address |
| "This browser has reached the current faucet claim limit" | Too many successful claims have come from your network/IP recently | Try again later, or from a different network |
| "The faucet is temporarily out of funds" (Depleted) | The faucet wallet is low on testnet sBTC | Try again later — this is refilled periodically |
| "The faucet is currently unavailable" | The faucet is disabled, misconfigured, or the node can't be reached | Try again shortly; if it persists, let the Sundial team know |

Address-related errors (invalid format, wrong network, no payment
credential, or a script/contract address) all mean the same practical thing:
paste a normal receive address copied from a testnet-mode wallet, not a stake
address, script address, or a mainnet address.

## Step 3 — Confirm you received your funds

Refresh the balance in your wallet extension (still on the Testnet/Preprod
network) — your testnet sBTC balance should reflect the grant amount shown
in the confirmation.

Note that this balance lives on Sundial's Midgard L2, not on Cardano's L1
chain, so it will not show up on general-purpose Cardano block explorers like
Cardanoscan. There isn't yet a public Midgard L2 block explorer during the
testnet phase — your wallet's balance, and the claim confirmation shown on
the faucet page, are the way to confirm a claim went through.

## Testnet network details

For reference, or if you're comparing notes with the Sundial team:

- **Faucet page:** `https://www.sundialprotocol.com/testnet/faucet`
- **Testnet address format:** `addr_test1…` (same as Cardano Preprod)
- **Testnet sBTC units:** displayed amounts are in sBTC; 1,000,000 of the
  underlying base unit ("lovelace") = 1 sBTC, matching how the faucet page
  formats the grant amount.
- **Per-claim limits:** each address has a cooldown before it can claim
  again, and there's a daily cap on claims from a given network/browser. The
  exact cooldown for your claim is always shown in the confirmation — these
  limits exist purely to keep the shared faucet available for everyone
  testing, not to restrict genuine testing.

You do not need to interact with the underlying node RPC directly to use the
faucet or hold testnet sBTC — the web page handles that for you. The faucet's
backing node lives at `https://sundial-node.testnet.sundialprotocol.com`, but
claim submission specifically requires a server-side credential the faucet
page holds on your behalf, so claims must go through the web page rather than
a direct API call.

## Explore the demo dashboard

Sundial's [dashboard](https://www.sundialprotocol.com/dashboard) shows what
the full Bitcoin-yield product looks like: connecting a wallet, depositing,
and tracking positions. On testnet it runs against **Bitcoin's testnet3
network**, using a separate Bitcoin wallet (not the Cardano-style wallet from
above) connected through the page's **Connect Wallet** button.

> The dashboard is an early-stage, test-only preview — transactions and
> balances there are simulated for demonstration purposes. Make sure any
> wallet you connect there is also set to testnet3, and never send real BTC
> to it.

This is a separate, optional flow from claiming faucet sBTC — you can use the
faucet on its own without ever connecting a wallet to the site.

## Sending your testnet sBTC to someone else

There isn't a "Send" button for this yet — the demo dashboard covers
depositing and tracking yield positions on **Bitcoin's testnet3 network**,
which is a separate flow from the testnet sBTC balance you just claimed on
Sundial's L2.

Paying another address with your testnet sBTC today means talking to
Sundial's L2 node directly, not just your wallet: your wallet extension
signs the transaction, but something has to look up your spendable testnet
sBTC and hand the signed transaction to Sundial's node, because a wallet's
built-in "Send" only knows about the regular Cardano network, not Sundial's
L2. That's a developer-facing flow today, not a point-and-click one — see
[api.md](./api.md#2-build-a-transaction-to-submit) if you or someone on your
team wants to build it.

## Safety reminders

- Testnet tokens (sBTC, and Bitcoin testnet3 coins) have **no monetary
  value**. Don't buy, sell, or trade them as if they did.
- Keep testnet wallets and seed phrases separate from any wallet holding real
  funds. Never enter a seed phrase you use for real assets into a testnet
  flow.
- If the faucet or dashboard behaves unexpectedly, that's useful testnet
  feedback — please report it to the Sundial team rather than assuming it's
  something you did wrong.
