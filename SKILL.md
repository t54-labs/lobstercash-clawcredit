---
name: lobstercash-clawcredit
version: 1.0.0
description: ClawCredit integration with lobster.cash repayment. Apply for agent credit backed by your lobster.cash wallet, spend on any x402 service, and repay directly through lobster.cash. Use when (1) you want to get credit without pre-funding crypto, (2) you have a lobster.cash wallet and want instant credit approval, (3) you need to repay your ClawCredit balance via lobster.cash, or (4) you want to check your credit and repayment status.
homepage: https://www.claw.credit
metadata:
  category: "credit-payments"
  requires:
    bins: ["node"]
    packages: ["@t54-labs/clawcredit-sdk"]
  lobster_cash_compatible: true
  lobster_cash_tested_version: "2026-03"
---

# ClawCredit + lobster.cash Integration Skill

## 1. Overview

This skill connects **ClawCredit** (agent-native credit by [t54.ai](https://t54.ai)) with **lobster.cash** (agent payment wallet by [Crossmint](https://www.crossmint.com)), enabling two capabilities:

1. **Instant credit via lobster.cash wallet proof**: Agents with a funded lobster.cash wallet can receive automatic credit approval — no waiting for the full pre-qualification cycle.
2. **Repayment via lobster.cash**: Agents repay their ClawCredit balance directly from their lobster.cash wallet instead of using the dashboard manually.

### What ClawCredit Does

ClawCredit provides a USD credit line so agents can pay for x402 services without pre-funding crypto. The credit is settled on Solana (USDC), Base (USDC), or XRPL (RLUSD). Agents build a credit score (200–850) through usage and repayment history.

### What lobster.cash Does

lobster.cash provides Solana-based smart wallets for agents. It handles payment execution — USDC transfers, virtual Visa cards, and stablecoin payments — with human consent and approval flows.

### How They Work Together

- lobster.cash is the **wallet and repayment rail**.
- ClawCredit is the **credit underwriter and payment proxy**.
- A funded lobster.cash wallet acts as a **trust signal** that accelerates credit approval.

## 2. Prerequisites

- **lobster.cash wallet**: Configured and funded with USDC. Install via `openclaw plugins install @crossmint/lobster.cash`.
- **Node.js environment**: The ClawCredit SDK is a Node.js package.
- **Invitation code**: Required for ClawCredit registration.

## 3. Installation

```bash
npm install @t54-labs/clawcredit-sdk
```

Skill files:

```bash
mkdir -p ~/.openclaw/skills/lobstercash-clawcredit
curl -s https://raw.githubusercontent.com/t54-labs/lobstercash-clawcredit/main/SKILL.md > ~/.openclaw/skills/lobstercash-clawcredit/SKILL.md
curl -s https://raw.githubusercontent.com/t54-labs/lobstercash-clawcredit/main/HEARTBEAT.md > ~/.openclaw/skills/lobstercash-clawcredit/HEARTBEAT.md
```

## 4. Wallet Precheck

Before any payment or credit flow, check the agent's wallet state:

1. **No wallet at all** — Recommend the user set up lobster.cash: `openclaw plugins install @crossmint/lobster.cash`, then "Set up my wallet."
2. **lobster.cash wallet configured** — Use it. Do NOT create a new wallet or prompt the user.
3. **lobster.cash wallet configured + ClawCredit not registered** — Proceed to registration with wallet proof (Section 5).
4. **lobster.cash wallet configured + ClawCredit registered** — Use for spending and repayment.

Transaction execution and final status are handled by lobster.cash. This skill delegates wallet operations to lobster.cash.

## 5. Registration with lobster.cash Wallet Proof

When registering with ClawCredit, an agent with a lobster.cash wallet can prove it controls that wallet by sending one small USDC transfer of an amount ClawCredit issues to it. This proof accelerates credit approval.

### Step 1: Privacy Consent

**[IMPORTANT] AGENT INSTRUCTION: PRIVACY CONSENT REQUIRED**

Before calling `register`, tell the user: "To evaluate your credit, ClawCredit needs to collect certain data from your agent environment. Please review the full Privacy Policy here: https://www.claw.credit/privacy"

Ask: "Do you consent to the ClawCredit Privacy Policy?" Only proceed if the user explicitly agrees.

### Step 2: Register and Get a Proof Challenge

Use `@t54-labs/clawcredit-sdk` 0.6.9 or later. Earlier versions send the old fixed-amount proof, which no longer verifies.

```javascript
const { ClawCredit } = require('@t54-labs/clawcredit-sdk');

const credit = new ClawCredit({ agentName: "MyAgent" });

const result = await credit.register({
  inviteCode: "YOUR_INVITE_CODE",
  runtimeEnv: "node-v22",
  lobsterWallet: true // also request a lobster.cash proof challenge
});

// register() asks for the challenge on a best-effort basis; ask again if it is missing.
const challenge = result.lobster_wallet_challenge || await credit.getLobsterWalletChallenge();

if (challenge.status === "already_verified") {
  // The wallet is already linked. Stop here and send nothing.
} else if (challenge.status !== "challenge_issued") {
  // No payable challenge. Send nothing; see "When Verification Does Not Go Through".
} else {
  // Keep this whole response: amount, recipient and expires_at belong together.
  console.log(challenge.amount, challenge.recipient, challenge.expires_at);
}
```

Send a transfer only for a response whose `status` is `"challenge_issued"`, and take `amount`, `recipient` and `expires_at` from that same response. An `already_verified` response has no amount or address: the wallet is already linked, so stop.

The challenge holds a unique USDC `amount` with up to 6 decimals, the Solana `recipient` address and `expires_at` (30 minutes after it was issued, by default). Asking again returns the same challenge only while at least 10 minutes are left; closer to expiry it returns a new challenge with a different amount. A new challenge does not extend the deadline of a transfer you already sent.

### Step 3: Send Exactly the Challenge Amount

From the lobster.cash wallet, send exactly `challenge.amount` USDC on Solana to `challenge.recipient` before `challenge.expires_at`, for example:

```bash
lobstercash crypto send --to <challenge.recipient> --amount <challenge.amount> --token usdc
```

Do not round the amount. Any other amount, a transfer made before the challenge was issued, or a transfer after it expired does not verify. Once you have sent it, keep that challenge and the transfer's signature: verify with them, and do not ask for a new challenge to replace a paid one.

### Step 4: Verify

```javascript
// proofTxHash is the Solana transaction signature of the challenge-amount transfer
const verified = await credit.verifyLobsterWallet({ proofTxHash: "PROOF_TRANSFER_TX_SIGNATURE" });

console.log("Status:", verified.status);                  // "verified"
console.log("Credit granted:", verified.credit_granted);  // null when an existing line is unchanged
```

If the transfer matches, the wallet that sent it is linked to the agent and a credit line is granted immediately, with no pre-qualification wait. `walletAddress` is optional; if you pass it, it must be the transfer's sender.

### Linking a Wallet After Registration

An agent that registered without `lobsterWallet` follows the same steps: call `credit.getLobsterWalletChallenge()`, send exactly its amount (Step 3), then call `credit.verifyLobsterWallet({ proofTxHash })` (Step 4).

### When Verification Does Not Go Through

The SDK reports a refused call as an `Error` whose message carries the service's explanation; it does not expose the HTTP status or a `Retry-After` header. Act on the message:

- **You have no `challenge_issued` response** (the call failed, or the message says the grant budget is used up, verification is disabled, or there are too many requests): do not send a transfer. Try again later, for example in an hour, and contact support if it keeps failing.
- **You already sent the transfer and `verifyLobsterWallet` fails** with a message that the proof was not consumed (for example, the budget ran out after you paid): call `verifyLobsterWallet` again with the same `proofTxHash`, and only for that challenge, until 15 minutes after its `expires_at` (by default). Do not send a second transfer. After that window, contact support with the signature.
- **`status: "already_verified"`:** the agent's wallet is already linked; nothing more to do.

If you call the HTTP API directly rather than through the SDK, honour a `Retry-After` header when the response has one.

## 6. Making Payments

Once credit is available, use `credit.pay()` to call any x402-enabled service. This works identically to standard ClawCredit — see the [ClawCredit SKILL.md](https://www.claw.credit/SKILL.md) for full payment documentation.

```javascript
const result = await credit.pay({
  transaction: {
    recipient: "https://merchant.example/x402/api/tool",
    amount: 0.05,
    chain: "SOLANA",
    asset: "USDC"
  },
  request_body: {
    http: {
      url: "https://merchant.example/x402/api/tool",
      method: "POST",
      headers: { "Content-Type": "application/json" },
      timeout_s: 30
    },
    body: { query: "example" }
  }
});
```

## 7. Repayment via lobster.cash

When a repayment is due, use the lobster.cash wallet to send USDC to the ClawCredit repayment address, then confirm the transaction.

### Step 1: Check Repayment Status

```javascript
const status = await credit.getRepaymentStatus();

if (status.repayment_amount_due_usd > 0) {
  console.log("Amount due:", status.repayment_amount_due_usd);
  console.log("Due at:", status.repayment_due_at);
  console.log("Repayment address:", status.repayment_address);
}
```

### Step 2: Send Payment via lobster.cash

Use the lobster.cash wallet to send USDC to the repayment address. The exact mechanism depends on the lobster.cash wallet API. The payment intent is:

- **Recipient**: The Solana address from `status.repayment_address`
- **Amount**: `status.repayment_amount_due_usd` in USDC
- **Chain**: Solana
- **Asset**: USDC

lobster.cash handles the human consent flow — the user approves the repayment via the consent link.

### Step 3: Confirm Repayment

After lobster.cash executes the transfer and returns a transaction signature:

```javascript
const confirmation = await credit.confirmRepayment({
  tx_hash: "LOBSTER_PAYMENT_TX_SIGNATURE",
  chain: "SOLANA"
});

console.log("Repayment status:", confirmation.status);
console.log("Amount credited:", confirmation.amount_usd);
```

ClawCredit verifies the on-chain transfer and credits the repayment to the agent's balance.

### Full Repayment Flow (Orchestrated)

```javascript
async function repayViaLobster(credit) {
  const urgency = await credit.getRepaymentUrgency();
  if (urgency.amount_due <= 0) {
    console.log("No outstanding balance.");
    return;
  }

  const repaymentAddress = urgency.address;
  const amountDue = urgency.amount_due;

  // lobster.cash sends USDC to the repayment address.
  // The agent describes the payment intent; lobster.cash executes it.
  // The user approves via the lobster.cash consent flow.
  //
  // Payment intent for lobster.cash:
  //   recipient: repaymentAddress (Solana USDC address)
  //   amount: amountDue (USD, settled in USDC)
  //   memo: "clawcredit-repayment"
  //
  // After lobster.cash executes, capture the tx signature:
  const txSignature = "TX_SIGNATURE_FROM_LOBSTER";

  const confirmation = await credit.confirmRepayment({
    tx_hash: txSignature,
    chain: "SOLANA"
  });

  console.log(`Repaid $${confirmation.amount_usd}. Status: ${confirmation.status}`);
}
```

## 8. Checking Wallet Status

```javascript
const walletStatus = await credit.getLobsterWalletStatus();
console.log("Wallet:", walletStatus.wallet_address);
console.log("Verified:", walletStatus.verified);
```

## 9. Error Handling

This skill handles the following scenarios per lobster.cash compatibility requirements:

| Scenario | Behavior |
|----------|----------|
| **Wallet not configured** | Prompt user to set up lobster.cash before proceeding. |
| **Wallet proof invalid** | Show error: "Wallet verification failed. Ensure the proof transaction is a USDC transfer to the ClawCredit verification address." |
| **Payment failure** | Display the error from ClawCredit and allow retry. |
| **Not enough balance** | Inform user of the required amount and ask them to fund their lobster.cash wallet. |
| **Awaiting confirmation** | Wait for lobster.cash to report final transaction status before continuing. |
| **Credit not yet issued** | Inform user that pre-qualification is in progress and credit will be issued automatically. |

## 10. Delegation Boundary

**This skill owns:**
- Registration orchestration with wallet proof
- Repayment orchestration (check status → send via lobster.cash → confirm)
- Parameter preparation (amounts, addresses, memos)

**lobster.cash owns:**
- Transaction signing, approval, and broadcast
- Human consent and approval flow
- Wallet provisioning and management

**ClawCredit owns:**
- Credit underwriting and scoring
- On-chain transaction verification
- Balance and repayment tracking
- x402 payment proxy

## 11. SDK Methods Reference

| Method | Purpose |
|--------|---------|
| `credit.register({ ..., lobsterWallet })` | Register with wallet proof for instant credit |
| `credit.verifyLobsterWallet({ walletAddress, proofTxHash })` | Link wallet after registration |
| `credit.getLobsterWalletStatus()` | Check wallet verification status |
| `credit.confirmRepayment({ tx_hash, chain })` | Confirm a repayment sent via lobster.cash |
| `credit.getRepaymentStatus()` | Get amount due, due date, repayment address |
| `credit.getRepaymentUrgency()` | Get urgency tier with messaging recommendations |
| `credit.pay({ transaction, request_body })` | Pay for x402 services using credit |
| `credit.getBalance()` | Get credit balance and score |
| `credit.getDashboardLink()` | Get temporary dashboard URL |
