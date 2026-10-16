# SIP-1: Monetization for Stellarium Resources

- **Status:** Draft
- **Author:** Christopher Poulsen ([@cjpoulsen](https://github.com/cjpoulsen))
- **Target networks:** Solana localnet, then devnet, then mainnet-beta
- **Payment asset:** SOL, priced in lamports (1 SOL = 10^9 lamports)
- **Identity:** Stellarium node Ed25519 keys
- **Settlement:** a centralized operator in version 1, replaced by a validator network in a later SIP
- **Reference implementation:** [stellarium-ts](https://github.com/qualletio/stellarium-ts) (`packages/stellarium-ts`, `apps/operator`, `programs/escrow_program`)

## 0. What is Stellarium

Stellarium is a decentralized API network. Nodes host Resources (named collections of typed functions), share them with their peers, and run RequestScript programs that call Resources anywhere on the network. See the [whitepaper](../WHITEPAPER.md) for the full design.

**Read more:**

- [https://github.com/qualletio/stellarium-ts](https://github.com/qualletio/stellarium-ts)
- [https://github.com/qualletio/requestscript-js](https://github.com/qualletio/requestscript-js)
- [https://github.com/qualletio/requestscript-js/blob/main/intro-guide.md](https://github.com/qualletio/requestscript-js/blob/main/intro-guide.md)

## 1. Summary

This SIP lets a Resource provider charge a fixed price, in lamports, for each successful call to a Resource function. A caller node funds a Solana escrow account tied to its node identity. Before calling a priced function, the caller node signs a payment voucher for that exact call. The provider verifies the voucher, executes the function, and submits the voucher to the Stellarium operator. The operator verifies it, rejects duplicates, and instructs the escrow program to pay 80% of the price to the provider and 20% to the protocol fee wallet.

A valid voucher is also the caller's permission to call a priced function. Providers need no API keys, accounts, or billing system.

## 2. Motivation

Developers who want to charge for an API today have to build sign-up, key management, usage tracking, and an integration with a payment processor. Stellarium already routes every call between nodes, so it knows exactly when a priced function is called and by whom. Attaching payment to the call removes that work from providers and gives callers one funding source for every API on the network.

## 3. Goals and non-goals

### 3.1 Goals

- A provider can set a fixed lamport price for each function of a Resource.
- A caller node pays every provider called by a RequestScript program from one escrow account.
- A caller node pays only for calls it signed a voucher for, and each voucher authorizes exactly one call with exactly one set of arguments.
- A provider claims payment only for calls that executed successfully.
- A provider is paid even if the caller node goes offline after receiving the result.
- Prices are public before any call is made.
- A voucher is settled at most once.

### 3.2 Non-goals for version 1

- **Decentralized settlement.** Version 1 uses a single operator (Section 5). A validator network is future work (Section 12).
- **Permissionless relays.** An earlier draft of this SIP proposed permissionless relays that batched receipts on Base. This draft replaces relays with the operator model.
- **Stablecoins.** Version 1 uses SOL only. An SPL stablecoin escrow is an open question (Section 11).
- **Disputes and arbitration.** Version 1 has no challenge window or arbitration. Provider honesty is addressed by the trust model in Section 9.
- **End-user payments.** Escrow accounts belong to nodes. How a node charges its own end users is outside this SIP.
- Subscriptions, free quotas, input-dependent prices, and metered prices.
- Nested calls. Resource functions cannot call other Resources through Stellarium, so every priced call is made directly by the caller node's interpreter.

## 4. Terminology

| Term | Meaning |
| :--- | :--- |
| **Caller node** | The node that runs a RequestScript program and pays for the priced calls it makes. |
| **Provider node** | The node that hosts a Resource and executes its functions. |
| **Node identity** | A node's Ed25519 key pair (`SIGNING_PUBLIC_KEY`, `SIGNING_PRIVATE_KEY`). The raw 32-byte public key is also the node's Solana address. |
| **Operator** | The service that creates escrow accounts and settles vouchers. In version 1 its key is the escrow program's authority. |
| **Escrow account** | A Solana program-derived account (PDA) that holds a caller node's SOL. |
| **Payment manifest** | The public price of a function: `{ costLamports, recipientAddress }`. |
| **Payment voucher** | A caller node's signature authorizing one payment for one specific call. |
| **Receipt** | The operator's record of a settled voucher and its Solana transaction. |
| **Protocol fee** | The share of each payment sent to the protocol fee wallet: 20% (2,000 basis points), rounded down. |

## 5. The operator model

### 5.1 Why an operator

Version 1 uses one operator instead of a network of relays for four reasons:

1. **Simplicity.** One service with one database can reject duplicate vouchers with a unique index. A relay network has to solve duplicate submission across independent parties.
2. **Fees.** The operator pays Solana transaction fees and escrow rent, so caller and provider nodes need no SOL for gas.
3. **Speed of iteration.** Payment rules live mostly in the operator, so they can change without redeploying the on-chain program.
4. **A clear path forward.** The operator has a narrow, well-defined job: verify a voucher and submit a withdrawal. That job is what a validator network will take over later.

### 5.2 Responsibilities

The operator:

- creates escrow accounts on request (`POST /v1/escrow`);
- verifies submitted vouchers (signature, expiry, uniqueness);
- submits withdrawals from the escrow program;
- records a receipt for every settled voucher;
- pays Solana transaction fees and escrow rent.

The operator does **not** host Resources, run RequestScript, route calls, or set prices.

### 5.3 What callers trust the operator with

In version 1 the escrow program accepts any withdrawal signed by the operator. It does not check the caller's voucher. The operator is also the program's upgrade authority. Callers are therefore trusting the operator to:

- withdraw only amounts covered by a valid voucher;
- send funds only to the voucher's recipient;
- not upgrade the program in a way that harms escrow holders.

Section 10 lists the changes that reduce this trust.

## 6. Prices

A provider attaches a payment manifest to any function it wants to charge for:

```typescript
interface StellariumResourceFunctionPaymentManifest {
  costLamports: number;      // price per successful call, a positive safe integer
  recipientAddress: string;  // base58 Solana address that receives the provider's share
}
```

The manifest is published next to the function signature in `GET /v1/resources`, and in the signed Resource announcements a node sends to its peers. A function without a manifest is free.

A price is fixed for every call to the function, whatever its arguments. A provider changes a price by updating its Resource definition and restarting its node, which re-announces its Resources to its peers. A voucher signed under the old price remains payable until it expires.

The recipient receives 80% of `costLamports`. Providers who want to receive a particular net amount must set their price accordingly.

## 7. Specification

### 7.1 Escrow program

**Status: Implemented** (`programs/escrow_program`).

**Account address.** `PDA(["escrow", authority, user], programId)`, where `authority` is the operator's key and `user` is the caller node's Solana address.

**Account layout** (73 bytes):

| Offset | Size | Field |
| :--- | :--- | :--- |
| 0 | 8 | Discriminator, ASCII `ESCROW01` |
| 8 | 32 | User (caller node address) |
| 40 | 32 | Authority (operator address) |
| 72 | 1 | Canonical bump seed |

**Instructions.**

| Data | Name | Accounts | Effect |
| :--- | :--- | :--- | :--- |
| `[0]` | Initialize | payer (writable, signer), user, authority (signer), escrow (writable), System Program | Creates the escrow account. The payer covers rent. The user does not sign. Accepts an address that was already sent a small amount of SOL, so nobody can block setup by sending dust first. |
| `[1, amount: u64 LE]` | Withdraw | authority (signer), escrow (writable), recipient (writable), fee recipient (writable) | Debits `amount` from the escrow. Credits `amount * 2000 / 10000` (rounded down) to the fee recipient and the rest to the recipient. Fails if the escrow would drop below its rent-exempt reserve. |

The authority and the fee recipient are compiled into the program. Custom errors: `0` authority not configured, `1` wrong authority, `2` fee recipient not configured, `3` wrong fee recipient.

**Funding.** A caller node funds its escrow with an ordinary SOL transfer to the escrow address. No instruction is needed.

### 7.2 Payment voucher

**Status: Implemented** (`packages/stellarium-ts/src/resource/invoker.ts`).

```typescript
interface PaymentVoucher {
  requestId: string;        // UUID
  invocationId: string;     // UUID, unique per call
  amount: number;           // lamports, must equal the manifest's costLamports
  recipientAddress: string; // must equal the manifest's recipientAddress
  expiresAt: string;        // ISO 8601, 2 minutes after issue
  publicKey: string;        // caller node's SPKI DER public key, base64url
  signature: string;        // Ed25519 signature, base64url
}
```

The signature covers this object, in this key order:

```
{ requestId, invocationId, amount, recipientAddress, expiresAt,
  resourcePath, resourceName, parameters, publicKey }
```

`resourcePath`, `resourceName`, and `parameters` come from the call, not the voucher. Because the call's arguments are signed, a voucher cannot be reused for a different call. The signed bytes are `JSON.stringify` of the object above.

**Required change:** `JSON.stringify` output depends on key order and number formatting, so it is fragile to verify across languages. Before mainnet, this SIP requires a canonical encoding (RFC 8785 JSON Canonicalization Scheme) and published test vectors.

### 7.3 Caller node behavior

When a program calls a function that has a payment manifest, the caller node:

1. Rejects the call if `costLamports` exceeds the node's spending limit. The call fails with an `error` value inside the program, and the rest of the program continues.
2. Signs a voucher for the exact Resource, function, and arguments.
3. Sends `POST {baseUrl}/v1/run/resource` with the call and the voucher.

A function that has no manifest is called without a voucher.

**Current limitation:** the spending limit is a fixed 10,000 lamports per call, and each voucher gets a new random `requestId`. See Section 7.7.

### 7.4 Provider node behavior

On `POST /v1/run/resource`, the provider node:

1. **MUST** reject the call with `402 Payment Required` if the function has a payment manifest and the request carries no voucher. **Not yet implemented:** today a priced function can be called for free by leaving the voucher out.
2. **MUST** verify the voucher's signature against the call it received.
3. **MUST** reject the voucher if `amount` or `recipientAddress` differs from its own manifest, or if it has expired. **Not yet implemented.**
4. Executes the function.
5. Submits the voucher, with the call's `resourcePath`, `resourceName`, and `parameters`, to `POST {OPERATOR_URL}/payment` **only if** execution succeeded. A function that throws produces no claim.
6. Returns the result to the caller without waiting for settlement.

### 7.5 Operator API

**Status: Implemented** (`apps/operator`).

`POST /v1/escrow`, body `{ "user": "<caller node Solana address>" }`. Creates the caller node's escrow account and returns `{ escrowAddress, status, signature, blockhash, lastValidBlockHeight }`, where `status` is `confirmed`, `unconfirmed`, or `existing`. The operator pays the rent, so this route MUST require an operator bearer token to stop anyone from draining the operator with escrow creations. **Not yet implemented:** the operator's README describes the token, but the current code does not check it.

`POST /v1/payment`, body `{ voucher, resourcePath, resourceName, parameters }`. The operator:

1. Rejects the voucher if a receipt already exists for its `(requestId, invocationId)`.
2. Verifies the Ed25519 signature over the payload in Section 7.2.
3. Rejects the voucher if it has expired.
4. Derives the escrow user from the voucher's public key, so the payer is always the node that signed.
5. Checks that the escrow holds enough SOL above its rent reserve.
6. Simulates, signs, and submits the withdrawal, then records the receipt and transaction signature.

The caller does not choose the escrow or the amount; both come from the signed voucher.

| Status | Meaning |
| :--- | :--- |
| 200 | Settled and confirmed |
| 202 | Submitted, confirmation uncertain. The client must check the returned transaction signature before retrying. |
| 400 | Malformed request or invalid public key |
| 401 | Invalid signature, or (once implemented) missing operator token on `/escrow` |
| 404 | Escrow account not found |
| 409 | Duplicate voucher, insufficient escrow balance, or a settlement for this voucher already in progress |
| 410 | Voucher expired |
| 422 | Simulation or on-chain execution failed |
| 502 | Solana RPC failure before submission |

### 7.6 Settlement timing

Vouchers expire 2 minutes after they are issued, and the operator rejects an expired voucher. A provider therefore has to submit within that window. That is long enough for one call, and too short for a provider that queues claims or goes briefly offline. **Proposed:** separate the *execution* deadline (the provider must start executing before `expiresAt`) from the *claim* deadline (the provider must submit within 24 hours).

### 7.7 Per-request budgets

**Status: Proposed.** RequestScript has no loops, so a caller node can compute a program's maximum cost before running it: the sum of `costLamports` over every priced call expression in the source. A caller node SHOULD:

1. Compute the maximum cost and reject the program if it exceeds the node's budget for one request.
2. Use a single `requestId` for every voucher in the program, so the operator and provider can group a request's calls.

## 8. Permissioning

A valid, funded voucher is the caller's permission to call a priced function. A provider MAY also publish an access manifest that limits callers by node public key:

```typescript
interface AccessManifest {
  allow?: string[]; // if present, only these caller node public keys may call
  deny?: string[];  // these caller node public keys may never call
}
```

For priced calls the provider identifies the caller from the voucher's public key. Access manifests are enforced by the provider node only. They are specified here so that callers can see in advance whether they may call; the details are left to a future SIP.

## 9. Security considerations

| Threat | Version 1 mitigation | Remaining risk |
| :--- | :--- | :--- |
| A voucher is replayed | The operator rejects duplicate `(requestId, invocationId)` pairs | Replay protection lives in the operator's database, not on chain |
| A voucher is reused for a different call | The call's resource, function, and arguments are signed | None known |
| A provider overcharges | The caller signs the exact amount and checks it against its limit | None known |
| A provider claims for a call that failed or returned bad data | The interpreter checks the return type; claims expire after 2 minutes | The caller cannot contest a claim. Provider honesty is assumed. |
| A caller refuses to pay after receiving the result | The voucher is signed before the call | None |
| A priced function is called without a voucher | **None today** (Section 7.4) | Free use of priced functions |
| A peer redirects a Resource to its own node | **None today** | Calls and vouchers go to an impostor. Requires signed Resource ownership (whitepaper Section 9). |
| The operator moves escrowed funds without a voucher | **None:** the operator is trusted | Section 10 |
| The operator is offline | None | Providers cannot be paid; caller funds are locked |
| Caller funds are stuck | **None today:** there is no withdraw instruction for the caller | Section 10 |
| The operator learns call contents | None | Receipts store the full arguments |

## 10. Reducing operator trust

These changes are in scope for version 1, in priority order:

1. **Caller refund.** Add an instruction that lets the escrow user, signing with its node key, withdraw its own unspent balance after a notice period. Without it, a caller's funds can only leave the escrow through the operator.
2. **On-chain voucher verification.** Have the withdraw instruction verify the caller's Ed25519 voucher signature through Solana's Ed25519 signature verification program, and record settled `invocationId`s on chain. The operator can then only move funds that a caller authorized, to the recipient the caller signed for.
3. **Argument hashing.** Store a hash of the call's arguments in receipts instead of the arguments themselves.
4. **Upgrade authority.** Move the program's upgrade authority from the operator's hot key to a multisig, then to governance.

## 11. Economics

| Item | Paid by |
| :--- | :--- |
| Function price | Caller node, from escrow |
| Protocol fee, 20% of the price | Deducted from the price |
| Solana transaction fee, about 5,000 lamports per settlement | Operator |
| Escrow account rent | Operator |

Settling one call per transaction costs the operator about 5,000 lamports per call. For any function priced near or below that, the protocol fee does not cover settlement. **Proposed:** batch settlement, in which one transaction pays one provider for many vouchers. This requires the escrow program to accept several amounts per instruction.

The protocol fee rate, recipient, and use of funds are governance decisions. They should become parameters set through the SIP process rather than constants compiled into the program.

### Open questions

1. **Price stability.** Prices in lamports change in dollar terms as SOL's price moves. Should version 2 add an SPL stablecoin escrow, or prices quoted in USD and converted at settlement with an oracle?
2. **Operator fee coverage.** Should the operator be allowed a minimum price, so that every settlement at least covers its transaction fee?
3. **Claim window.** Is 24 hours the right claim deadline (Section 7.6)?

## 12. Future work

- **Validator network.** Replace the single operator with a set of validators that verify vouchers and submit settlements, so no single party controls payment. Version 1 should keep the operator's job narrow (verify, deduplicate, submit) so validators can take it over unchanged.
- **Disputes.** Caller acceptance receipts and a challenge window for provider claims.
- **Batch settlement** (Section 11).
- **Contracts as priced Resources.** A published contract could carry its own price on top of the Resources it calls.
