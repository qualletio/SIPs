# SIP-1: USDC Monetization for Stellarium

- **Status:** Draft
- **Target networks:** Base Sepolia, then Base Mainnet
- **Payment asset:** Base USDC
- **Identity:** externally owned accounts (EOAs) only in version 1

## 0\. What is Stellarium

Stellarium is a decentralized API network that uses RequestScript to execute http requests to servers. Nodes define Resources, share them across the network, and allow users to call them.

**Read more:**

- [https\://github.com/qualletio/stellarium-ts](https://github.com/qualletio/stellarium-ts)
- [https\://github.com/qualletio/requestscript-js](https://github.com/qualletio/requestscript-js)
- [https\://github.com/qualletio/requestscript-js/blob/main/intro-guide.md](https://github.com/qualletio/requestscript-js/blob/main/intro-guide.md)

## 1\. Summary

This RFC adds decentralized, post-execution payment for Stellarium Resources and individual Resource functions. A caller deposits USDC into a reusable Stellarium balance, reserves a maximum budget for a top-level RequestScript request, and signs an authorization with its EOA. Providers execute successfully before making a claim. A failed execution produces no payable receipt.

Every successful invocation is represented by a separate signed receipt. Permissionless relays batch receipts on Base. Each receipt specifies the relay that may submit it and the relay fee it earns. The provider's gross advertised price pays the provider, the relay, and a dynamically calculated protocol fee; the provider's net amount may be zero if fees consume the gross amount.

Providers may claim without caller cooperation, but all claims remain challengeable for one hour. Disputes use randomly selected, USDC-staked arbitrators and signed evidence stored encrypted off-chain. Resource and relay discovery remain peer-hosted: Stellarium nodes exchange signed records directly and no central index controls discovery, pricing, access, or settlement.

## 2\. Goals

- Let each Resource owner set a fixed USDC price for its Resource or individual functions.
- Pay every provider in a multi-provider RequestScript request from one caller-authorized budget.
- Charge only for successful function execution.
- Allow settlement if a caller receives a result and refuses to cooperate.
- Keep resource discovery, relay operation, and settlement permissionless.
- Make payment and pricing terms verifiable before execution.
- Preserve request and response confidentiality in the normal case; on-chain data contains hashes and payment metadata only.
- Provide external-wallet support for browser clients.

## 3\. Non-goals for version 1

- Subscriptions, free quotas, input-sensitive prices, and metered prices.
- Smart-account and ERC-1271 signature support.
- Provider bonds. A future version should require a slashing bond for unilateral provider claims.
- Subjective reputation attestations.
- A centralized catalogue, payment processor, custody wallet, or relay operator.

## 4\. Terminology

| Term               | Meaning                                                                                                                          |
| :----------------- | :------------------------------------------------------------------------------------------------------------------------------- |
| Caller             | EOA that owns the USDC balance and signs a request authorization.                                                                |
| Provider           | Resource owner that executes a function and signs its execution receipt.                                                         |
| Payment identity   | EOA that signs the Resource payment manifest. It declares the recipient and receipt-signing EOAs; all three may be the same EOA. |
| Relay              | Permissionless EOA that submits a valid batch of receipts to Base.                                                               |
| Gross price        | Fixed USDC amount advertised for one successful invocation, before relay and protocol fees.                                      |
| Request budget     | USDC reserved from the caller's reusable balance for one top-level request.                                                      |
| Execution receipt  | Provider-signed record that binds a successful invocation to a request, quote, hashes, fees, and an assigned relay.              |
| Acceptance receipt | The caller's EIP-712 signature over a valid execution receipt after receiving its result.                                        |
| Fallback claim     | Settlement using only a valid provider-signed execution receipt when no caller acceptance receipt exists.                        |

## 5\. Trust and security model

The settlement contracts enforce funds, signatures, expiry, nonces, duplicate protection, budget limits, and deterministic fee calculation. They cannot prove that an arbitrary API output is semantically correct or that a network response reached a caller. The dispute process addresses those claims using evidence and arbitration.

The protocol deliberately separates the normal and fallback paths:

1. The normal path has the caller accept a provider-signed execution receipt after receiving a result.
2. The fallback path permits a provider to claim against the reserved budget without caller acceptance.

Both paths have the same one-hour challenge period. This is necessary because both are disputable and version 1 has no provider bond from which an already-paid claim could be recovered. Funds stay locked until the challenge period expires or a dispute resolves.

## 6\. Identity and signed data

All version-1 signatures use EIP-712 typed data with the Base chain ID and verifying-contract address in the domain separator. Base Mainnet is chain ID `8453`; Base Sepolia is `84532`. See [Base's `eth_chainId` reference](https://docs.base.org/base-chain/api-reference/ethereum-json-rpc-api/eth_chainId).

Every signed object includes:

- `protocolVersion`
- `chainId`
- `verifyingContract`
- `issuedAt`
- `expiresAt`, when it is time-limited
- a unique nonce or sequence number

The SDK canonicalizes RequestScript source and arguments before hashing. It signs hashes of canonical values, never untrusted string interpolation. The reference implementation must publish the canonicalization algorithm and test vectors before Mainnet deployment.

### 6.1 Payment manifest

The existing `GET /v1/resources` response gains a signed `monetization` field for a Resource. It is signed by the `paymentIdentity` EOA and contains:

```typescript
interface MonetizationManifestV1 {

version: 1;

paymentIdentity: Address;

paymentRecipient: Address;

receiptSigner: Address;

evidenceEncryptionPublicKey: Hex;

chainId: 8453 | 84532;

usdc: Address;

quoteRegistryId: Hex;

functions: Array\<{

    name: string;

    grossPriceUsdc: bigint; // USDC's 6-decimal atomic units

    quoteId: Hex;

    issuedAt: number;

    expiresAt: number; // no later than one hour after issuedAt

}\>;

signature: Hex;

}
```

The payment identity may declare the same EOA for all three payment roles, but separate keys are supported from the beginning. Nodes verify the signature, time range, and on-chain key registration before exposing a manifest to callers.

### 6.2 On-chain price registry

`ResourceRegistry` anchors the payment identity, currently authorized recipient and receipt-signing keys, Resource identifier, and the current quote commitment. The signed peer-hosted manifest provides convenient discovery; the registry gives callers an on-chain reference for the same provider and price terms.

Quotes are fixed-price and expire hourly. A receipt is valid only if its quote was active when the function executed and matches the registry commitment. Providers may publish newer quotes before old quotes expire, but cannot rewrite an existing quote.

### 6.3 Request authorization

Before execution, the caller opens a request budget and signs a `RequestAuthorization`:

```typescript
interface RequestAuthorizationV1 {
  requestId: Hex;

  caller: Address;

  budgetUsdc: bigint;

  requestHash: Hex;

  expiresAt: number;

  nonce: bigint;
}
```

`openRequestBudget` transfers the requested USDC amount from the caller's reusable available balance into a request-specific reserved balance. The caller authorizes only a total budget, never a predefined provider list or individual price caps. This allows a Resource to invoke another paid Resource while keeping all claims bounded by the original request budget.

Unused money returns to the caller's available Stellarium balance when the request expires or closes. The caller can withdraw any unreserved available balance immediately.

### 6.4 Execution and acceptance receipts

interface ExecutionReceiptV1 {

receiptId: Hex;

requestId: Hex;

providerPaymentIdentity: Address;

paymentRecipient: Address;

receiptSigner: Address;

resourceId: Hex;

functionNameHash: Hex;

parametersHash: Hex;

responseHash: Hex;

quoteId: Hex;

grossPriceUsdc: bigint;

relay: Address;

relayFeeUsdc: bigint;

executedAt: number;

providerSignature: Hex;

}

The receipt contains no request arguments or output. It is signed only after successful execution. The receipt must be submitted within 30 minutes of `executedAt`.

The caller may add an EIP-712 acceptance signature after receiving the response. The provider sends the complete record to its assigned relay over the peer network. The assigned relay is selected automatically by the provider node from signed, peer-discovered relay advertisements; neither callers nor providers configure a relay manually.

## 7\. Request and settlement flow

sequenceDiagram

participant C as Caller wallet and SDK

participant N as Stellarium node

participant P as Resource provider

participant R as Selected relay

participant S as Base settlement contracts

C-\>\>S: Deposit USDC; reserve request budget

C-\>\>N: Signed request authorization and RequestScript source

N-\>\>P: Invoke paid function with request authorization

P-\>\>P: Execute successfully

P--\>\>N: Response plus provider-signed receipt

N--\>\>C: Response and receipt

C-\>\>P: Optional acceptance signature

P-\>\>R: Signed receipt and optional acceptance

R-\>\>S: Batch claims

S-\>\>S: Lock each claim for one-hour challenge period

S--\>\>R: Release relay fee after challenge period

S--\>\>P: Release provider net after challenge period

### 7.1 Invocation rules

1. The caller discovers a valid, hourly signed Resource manifest from a peer and verifies its registry anchor.
2. The wallet deposits USDC if needed, then reserves a request budget in `StellariumBalanceVault`.
3. The caller signs `RequestAuthorization` and submits it with the RequestScript request.
4. Every paid local or remote invocation resolves the provider's quote before execution. A local invocation serving an external caller follows the same payment process as a forwarded invocation.
5. A provider executes the function. A failure emits no execution receipt and consumes no request budget for that provider.
6. A successful provider returns the response and its signed receipt. The caller SDK normally signs acceptance after validating the response hash.
7. The provider or node forwards the receipt to the assigned relay. A relay batches compatible claims to reduce gas cost.

Nested Resource calls may consume the same request budget. Contract settlement accepts claims in deterministic on-chain order until the budget is exhausted. The SDK should reserve sufficient headroom and present the total authorized budget clearly before the caller signs.

### 7.2 Claim validation

Before accepting a claim, `Settlement` automatically rejects:

- invalid caller, provider, or acceptance signatures;
- an unregistered or unauthorized provider, recipient, or receipt signer;
- a quote whose gross price does not equal the receipt price;
- an expired quote, request authorization, or submission window;
- duplicate `receiptId`, quote use, or nonce;
- a receipt whose requested amount exceeds the remaining request budget;
- a relay submission from an EOA other than the receipt's assigned relay;
- stale Chainlink oracle data older than 15 minutes.

Valid claims reserve the gross amount in the request budget and start the one-hour challenge window. An uncontested fallback claim pays automatically when the window expires. An acceptance receipt follows the same window because either party may dispute it.

## 8\. Fees

For each receipt:

providerNet \= max(0, grossPrice \- relayFee \- protocolFee)

The receipt declares `relayFee`; the assigned relay receives it. `Settlement` calculates `protocolFee` at batch settlement time from the Base execution and data fee, allocated equally across all receipts in that batch, and an ETH/USD price from Chainlink. The contract rejects Chainlink data older than 15 minutes.

The protocol fee is split 80/20:

- 80% enters a DAO-controlled relay and operations reserve, which funds relay gas shortfalls, randomness requests, and arbitration administration.
- 20% remains in the DAO treasury.

This scheme may yield a zero provider net. The contract permits that result as specified. A later protocol version may choose proportional-by-gross fee allocation rather than equal-per-receipt allocation; this version deliberately uses equal allocation because it is easy to audit.

The fee module uses an oracle interface. The initial deployment requires Chainlink. Pyth is a compatible future implementation of the same interface, rather than a silent fallback. Chainlink VRF v2.5 is currently available on Base Mainnet and Base Sepolia, and Base documents Pyth support on both networks. [Chainlink VRF changelog](https://docs.chain.link/changelog/vrf-v2-5), [Base Pyth guide](https://docs.base.org/cookbook/use-case-guides/finance/access-real-time-asset-data-pyth-price-feeds/)

## 9\. Disputes and arbitration

### 9.1 Grounds

The caller or provider may dispute an acceptance or fallback claim within one hour after on-chain claim submission. The initial arbitration scope is limited to:

- the Resource did not execute;
- the response did not conform to the advertised return type;
- the response was not delivered;
- the provider charged for another function or input.

The court does not decide whether an arbitrary API result is substantively correct.

### 9.2 Evidence

On-chain records expose only hashes and payment metadata. The request, response, and signed receipts are stored in encrypted peer-hosted evidence storage. Every active arbitrator publishes an encryption public key with its stake registration. After the randomness adapter selects a panel, each party releases the necessary encrypted evidence directly to each selected arbitrator.

Each party has 24 hours from panel selection to provide requested evidence. A party that misses the deadline loses the dispute. This process trades fully public evidence for limited disclosure to the selected panel.

### 9.3 Panel selection and outcomes

`ArbitrationCourt` obtains random words through the abstract `RandomnessAdapter`. The initial adapter deployment may use Chainlink VRF v2.5 on Base; adapters are replaceable only through governance and timelock.

Arbitrators stake USDC. They are sampled using stake weight multiplied by a transparent reputation multiplier capped at 3x. The court uses fresh random draws and new panels for appeals:

| Round            | Panel size |
| :--------------- | :--------- |
| Initial decision | 3          |
| Appeal 1         | 5          |
| Appeal 2         | 7          |

There are at most three rounds in total. The first decision can be appealed twice. Additional appeal fees are intentionally outside version 1 and must be added before any later expansion of appeals.

The court can pay the provider or refund the caller. Locked funds remain locked until final resolution. If the provider wins, the caller pays arbitration fees; if the caller wins, the provider pays them.

## 10\. Reputation

Reputation is derived only from verifiable, signed facts and on-chain outcomes:

- successful settlements;
- caller acceptance receipts;
- dispute outcomes;
- relay batch inclusion and completion.

`ReputationRegistry` stores event commitments and deterministically calculates role-specific scores. There are no subjective attestations. The score affects relay discovery ranking and relay selection, and applies the capped 3x multiplier in arbitrator selection.

New identities require sponsorship by three existing active participants. Each active sponsor may sponsor at most three identities in any rolling 30-day period. Sponsors incur no version-1 penalty for a sponsored identity's conduct. An implementation should make this deliberately limited admission rule visible to users; future versions may add sponsor accountability.

The exact score coefficients, eligibility threshold, and minimum USDC stake are deployment parameters that must be published in the deployment manifest and modified only through governance.

## 11\. Peer discovery

There is no central index. Existing peer bootstrap and Resource exchange remain the discovery mechanism.

`GET /v1/resources` publishes the Resource's ordinary signature-free interface plus its signed `monetization` manifest. Nodes relay valid manifests, quote updates, relay advertisements, evidence-location records, and reputation facts directly to known peers. A node verifies all records before caching or relaying them.

Relay advertisements are signed by the relay EOA and include:

- settlement contract and chain ID;
- relay EOA;
- supported receipt version;
- current availability and expiry;
- advertised operational parameters; and
- the relay's on-chain reputation reference.

The SDK and provider node select an eligible relay by a deterministic scoring function over signed advertisements, availability, and reputation. The resulting relay EOA is bound into the execution receipt, preventing front-running from redirecting the relay fee.

## 12\. Contracts

| Contract                            | Responsibility                                                                                                               |
| :---------------------------------- | :--------------------------------------------------------------------------------------------------------------------------- |
| `StellariumBalanceVault`            | Holds USDC deposits, available balances, request reservations, expiry release, and immediate withdrawal of unreserved funds. |
| `ResourceRegistry`                  | Anchors Resource payment identity, declared keys, recipient, and quote commitments.                                          |
| `Settlement`                        | Validates and batches receipts, reserves claims, calculates fees, applies challenge windows, and releases payments.          |
| `FeeOracle`                         | Calculates Base gas cost in USDC using a Chainlink ETH/USD source with 15-minute freshness.                                  |
| `RelayRegistry`                     | Registers USDC-staked relays and exposes the signed-advertisement identity binding.                                          |
| `ReputationRegistry`                | Records signed outcome facts and calculates transparent role scores.                                                         |
| `ArbitrationCourt`                  | Opens disputes, manages evidence deadlines, receives panel outcomes, handles appeals, and distributes final funds and fees.  |
| `RandomnessAdapter`                 | Abstract interface for a Base-compatible verifiable-randomness provider.                                                     |
| `ProtocolTreasury`                  | Holds the DAO portion of protocol fees and the relay and operations reserve.                                                 |
| `Governor` and `TimelockController` | Govern all upgradeable parameters and contract administration.                                                               |

All USDC transfers use safe ERC-20 wrappers. The implementation must use reentrancy guards, explicit state machines, and pull-payment withdrawal where applicable. Receipt IDs, request IDs, and quote IDs are permanently replay-protected.

## 13\. Governance and launch control

At launch, five publicly identified stewards operate a 3-of-5 multisignature. The multisig is the temporary proposer through a `TimelockController`; it cannot directly change production contracts.

- Ordinary parameter changes have a 48-hour timelock.
- Settlement, arbitration, registry, and implementation upgrades have a 7-day timelock.
- An emergency pause may stop new deposits and new claims for up to 24 hours. It cannot transfer user funds, alter receipts, resolve disputes, or block withdrawals of unreserved funds. Extending a pause requires the ordinary timelocked process.

Reputation governance replaces the launch multisig after 100 active participants exist. An active participant has nonzero reputation and at least one settled receipt in the preceding 90 days. Voting power derives from reputation. The deployment must publish the transition transaction and remove the multisig's proposer authority when the transfer completes.

OpenZeppelin's `TimelockController` is an appropriate implementation foundation because it exposes a minimum execution delay and can be self-administered after governance takes control. [OpenZeppelin governance documentation](https://docs.openzeppelin.com/contracts/5.x/api/governance)

## 14\. Node and SDK implementation plan

### 14.1 `stellarium-ts` node

1. Add versioned payment types, EIP-712 serializers, canonical hashing, and signature verification.
2. Extend the Resource representation and `GET /v1/resources` with validated `monetization` manifests.
3. Add a manifest cache, quote-expiry validation, peer relay-advertisement cache, and encrypted-evidence record exchange.
4. Extend resource invocation so local and forwarded paid calls receive a `PaymentExecutionContext` containing the request authorization, remaining budget, trace, and selected relay.
5. Refuse paid execution without a valid authorization, active quote, registered payment keys, and sufficient reserved budget.
6. Create an execution receipt only after `fn.exec` returns successfully. Do not create a receipt for exceptions, timeouts, or type-validation failures.
7. Propagate payment context on forwarded `/run` calls. Replace the current string-built forwarding request with a structured and safely encoded RequestScript request so parameter values and payment metadata cannot be injected into source text.
8. Return function output, execution receipt, provider evidence location, and quote data to the caller SDK.
9. Publish the receipt to the assigned relay and retain encrypted evidence until the challenge and appeal periods end.

### 14.2 TypeScript SDK

The SDK should provide:

- EIP-1193 injected-wallet and WalletConnect connection adapters; Coinbase Wallet works through either compatible connection path.
- Base Mainnet and Base Sepolia network configuration.
- USDC deposit, request-budget reservation, closing, and withdrawal flows.
- Resource and quote verification before invocation.
- a prominent total request-budget approval screen;
- automatic acceptance-receipt signing after response-hash validation;
- claim, challenge-window, dispute, and appeal status; and
- encrypted evidence creation and delivery to selected arbitrators.

The hosted wallet experience is a hosted user interface over external wallets. It never holds a private key or custody balance.

### 14.3 Solidity reference implementation

Implement each contract behind interfaces and deploy in this order on Base Sepolia:

1. USDC address configuration, `StellariumBalanceVault`, `ResourceRegistry`, and EIP-712 domain test harness.
2. `Settlement` with direct acceptance claims, then fallback claims and budget exhaustion tests.
3. `FeeOracle` with Chainlink freshness checks and Base fee-allocation tests.
4. `RelayRegistry`, deterministic off-chain selection test vectors, and receipt relay binding.
5. `ReputationRegistry` and sponsored-identity admission rules.
6. `RandomnessAdapter` and `ArbitrationCourt`, including all evidence-deadline and 3/5/7 appeal transitions.
7. `ProtocolTreasury`, Governor, timelock, launch multisig configuration, and governance-transfer checks.

## 15\. Rollout and verification

1. Publish protocol schemas, EIP-712 type hashes, canonicalization rules, deployment parameters, and signed test vectors.
2. Run end-to-end integration on Base Sepolia with independent caller, provider, relay, and arbitrator nodes.
3. Simulate failed calls, caller disconnects, duplicate claims, expired quotes, exhausted budgets, stale oracle data, relay front-running, evidence non-delivery, and every appeal outcome.
4. Obtain an independent smart-contract audit and remediate findings before holding Mainnet USDC.
5. Deploy the same versioned contracts to Base Mainnet only after the Sepolia rollout and audit pass.
6. Publish public steward identities, multisig threshold, timelock addresses, contract addresses, oracle address, parameters, and the governance-transfer condition.

## 16\. Deferred design work

- Add a provider bond and slashing policy for unilateral fallback claims.
- Define appeal fees and potentially an economic bond for appeals.
- Evaluate proportional fee allocation against equal-per-receipt allocation.
- Add Pyth through `FeeOracle` after an explicit governance vote and audit.
- Add smart-account / ERC-1271 signatures.
- Consider subscriptions, quotas, usage pricing, and provider-defined price functions only in a future protocol version.
- Reassess sponsor accountability and anti-Sybil protection as reputation governance becomes active.
