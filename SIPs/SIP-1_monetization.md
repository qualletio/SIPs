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

This RFC adds decentralized, post-execution payment for Stellarium Resources and individual Resource functions. A caller deposits USDC into a reusable Stellarium balance, reserves a maximum budget for a top-level RequestScript request, and signs an authorization with its EOA. The top-level RequestScript may invoke multiple Resource functions directly; Resource functions cannot invoke other Resource functions through Stellarium. Nested calls are not supported by the Stellarium protocol. Providers execute successfully before making a claim. A failed execution produces no payable receipt.

Every successful invocation is represented by a separate signed receipt. Permissionless relays batch receipts on Base. Each receipt specifies the relay that may submit it and the relay fee it earns. The provider's gross advertised price pays the provider, the relay, and a dynamically calculated protocol fee; the provider's net amount may be zero if fees consume the gross amount.

Providers may claim without caller cooperation, but all claims remain challengeable for one hour. Disputes use randomly selected, USDC-staked arbitrators and signed evidence stored encrypted off-chain. Resource and relay discovery remain peer-hosted: Stellarium nodes exchange signed records directly and no central index controls discovery, pricing, access, or settlement.

## 2\. Goals

- Let each Resource owner set a fixed USDC price for its Resource or individual functions.
- Pay every provider called directly by a top-level RequestScript request from one caller-authorized budget.
- Charge only for successful function execution.
- Allow settlement if a caller receives a result and refuses to cooperate.
- Keep resource discovery, relay operation, and settlement permissionless.
- Make payment and pricing terms verifiable before execution.
- Preserve request and response confidentiality in the normal case; on-chain data contains hashes and payment metadata only.
- Provide external-wallet support for browser clients.

## 3\. Non-goals for version 1

- Nested Resource calls are outside the Stellarium protocol; providers cannot delegate invocations or spend a caller's budget on further Resource calls.
- Subscriptions, free quotas, input-sensitive prices, and metered prices.
- Smart-account and ERC-1271 signature support.
- Provider bonds. A future version should require a slashing bond for unilateral provider claims.
- Subjective reputation attestations.
- A centralized catalogue, payment processor, custody wallet, or relay operator.

## 4\. Terminology

| Term               | Meaning                                                                                                                                |
| :----------------- | :------------------------------------------------------------------------------------------------------------------------------------- |
| Caller             | EOA that owns the USDC balance and signs a request authorization.                                                                      |
| Provider           | Resource owner that executes a function and signs its execution receipt.                                                               |
| Payment identity   | EOA that signs the Resource payment manifest. It declares the recipient and receipt-signing EOAs; all three may be the same EOA.       |
| Relay              | Permissionless EOA that submits a valid batch of receipts to Base.                                                                     |
| Gross price        | Fixed USDC amount advertised for one successful invocation, before relay and protocol fees.                                            |
| Request budget     | USDC reserved from the caller's reusable balance for one top-level request.                                                            |
| Execution receipt  | Provider-signed record that binds a successful invocation to a request, quote, hashes, fees, and an assigned relay.                    |
| Acceptance receipt | The caller's EIP-712 signature over a valid execution receipt after receiving its result.                                              |
| Fallback claim     | Settlement using a valid provider-signed execution receipt and execution-authority-signed invocation grant, without caller acceptance. |
| Invocation grant   | Permission signed by the caller-authorized execution authority for one direct Resource invocation in the top-level request.            |
| Manifest revision  | Immutable registry record of a Resource's payment terms, keys, quotes, and activation and retirement times.                            |

## 5\. Trust and security model

The settlement contracts enforce funds, signatures, expiry, nonces, duplicate protection, budget limits, and deterministic fee calculation. They verify that each invocation grant is signed by the caller-authorized execution authority and matches the receipt. They cannot prove that an invocation followed the authorized RequestScript control flow, that a provider-reported execution timestamp is truthful, that an arbitrary API output is semantically correct, or that a network response reached a caller. The dispute process addresses those claims using evidence and arbitration.

The protocol deliberately separates the normal and fallback paths:

1. The normal path has the caller accept a provider-signed execution receipt after receiving a result.
2. The fallback path permits a provider to claim against the reserved budget without caller acceptance, but still requires a valid invocation grant signed by the execution authority.

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
  resourceId: Hex;
  manifestRevision: bigint;
  quoteRegistryId: Hex;
  functions: []{
    name: string;
    grossPriceUsdc: bigint; // USDC's 6-decimal atomic units
    quoteId: Hex;
    issuedAt: number;
  };
  signature: Hex;
}
```

The payment identity may declare the same EOA for all three payment roles, but separate keys are supported from the beginning. Nodes verify the signature, current Resource manifest commitment, and on-chain key registration before exposing a manifest to callers. Quotes have no `expiresAt` field and remain active indefinitely unless that Resource's payment manifest is updated at node startup.

### 6.2 On-chain price registry

`ResourceRegistry` anchors the payment identity, currently authorized recipient and receipt-signing keys, Resource identifier, and the current quote commitment. The signed peer-hosted manifest provides convenient discovery; the registry gives callers an on-chain reference for the same provider and price terms.

Quotes are fixed-price and do not expire with time. They expire only when the provider node updates that Resource's payment manifest at node startup and anchors the new manifest and quote commitments in `ResourceRegistry`. The update retires all quotes from the previous payment manifest for that Resource and publishes new quote IDs; quotes for other Resources are unaffected. Restarting a node with an unchanged payment manifest preserves its existing quotes and quote IDs. Payment manifest updates and quote replacement occur only at node startup.

Each Resource has monotonically increasing, immutable manifest revisions. The registry retains each revision's manifest commitment, payment identity, recipient, receipt signer, function-price commitments, quote IDs, and activation and retirement timestamps. Activating a new revision atomically retires the previous revision. Activation and retirement use registry transaction block timestamps; an active interval includes its activation timestamp and excludes its retirement timestamp. A receipt identifies its manifest revision and must match that revision's Resource, function, quote, price, recipient, and receipt signer. Validation uses the historical revision covering `executedAt`, rather than the Resource's current keys. Routine key rotation therefore does not invalidate receipts for earlier valid executions.

Retiring a quote prevents its use for subsequent successful executions; receipts for executions completed while it was active remain eligible for settlement within the existing request-authorization and receipt-submission deadlines. Providers cannot rewrite an existing quote. Quote IDs identify immutable terms and may be referenced by any number of distinct invocations and receipts. A quote is never consumed by settlement.

Startup reconciliation compares canonical payment configuration, excluding generated signatures, timestamps, revision numbers, and quote IDs. An unchanged configuration reuses the registered revision and quote IDs. Changed configuration creates one new revision and new quote IDs for that Resource. The node waits for registry confirmation before advertising the new revision or accepting paid invocations under it.

### 6.3 Request authorization

Before execution, the caller signs a `RequestAuthorization` and opens a request budget bound to it:

```typescript
interface RequestAuthorizationV1 {
  requestId: Hex;
  caller: Address;
  budgetUsdc: bigint;
  requestHash: Hex;
  executionAuthority: Address; // EOA authorized to grant direct invocations in this request
  issuedAt: number;
  expiresAt: number;
  nonce: bigint;
}
```

`openRequestBudget` transfers the requested USDC amount from the caller's reusable available balance into a request-specific reserved balance. The caller authorizes only a total budget, never a predefined provider list or individual price caps. The execution authority may issue grants only for Resource invocations reached directly by the top-level RequestScript interpreter. Providers receive permission to execute their specific invocation and cannot authorize additional Resource calls. The SDK displays the execution authority before signing. Disclosure of the request authorization alone does not grant permission to spend its budget.

Budget opening binds the caller, request ID, authorization hash, budget, request hash, execution authority, and expiry on-chain. The caller's authorization nonce is consumed once at budget opening; the same bound authorization is then reused by all valid invocations in that request. A nonce cannot open another budget, and an existing request ID cannot be rebound to a different authorization.

Unused money returns to the caller's available Stellarium balance when the request expires or closes. The caller can withdraw any unreserved available balance immediately.

### 6.4 Execution and acceptance receipts

```typescript
interface ExecutionReceiptV1 {
  receiptId: Hex;
  requestId: Hex;
  authorizationHash: Hex;
  invocationId: Hex;
  invocationGrantHash: Hex;
  manifestRevision: bigint;
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
```

The receipt contains no request arguments or output. It is signed only after successful execution and output-type validation. `executedAt` is the provider's asserted successful completion time, not a trusted clock attestation. `receiptId` is the hash of the protocol domain, `requestId`, and `invocationId`; changing a receipt's response, relay, or signature cannot create another payable receipt for the same invocation. Both acceptance and fallback claims require the execution-authority-signed invocation grant defined in section 6.5.

At submission, the contract requires the budget to remain open and the request authorization to remain unexpired, with `authorization.issuedAt <= executedAt <= block.timestamp` and `block.timestamp - executedAt <= 30 minutes`. The referenced manifest revision must cover `executedAt`. These are deterministic consistency checks, not proof of when execution occurred. Backdating to use retired quotes or keys, or falsifying completion time to satisfy a submission deadline, is disputable. Caller acceptance does not waive that dispute. No receipt is payable if the actual successful completion occurred after the quote retired.

The caller may add an EIP-712 acceptance signature after receiving the response. The provider sends the complete record to its assigned relay over the peer network. The assigned relay is selected automatically by the provider node from signed, peer-discovered relay advertisements; neither callers nor providers configure a relay manually.

### 6.5 Direct invocation grants

Every paid invocation has an EIP-712 `InvocationGrant` signed by the request's `executionAuthority` before dispatch. In addition to the common signed-object fields from section 6, it contains:

```typescript
interface InvocationGrantV1 {
  requestId: Hex;
  authorizationHash: Hex;
  invocationNonce: Hex; // unique within this request
  providerPaymentIdentity: Address;
  resourceId: Hex;
  manifestRevision: bigint;
  functionNameHash: Hex;
  parametersHash: Hex;
  quoteId: Hex;
  grossPriceUsdc: bigint;
  expiresAt: number; // no later than the request authorization expiry
}
```

`invocationId` is the EIP-712 typed-data hash of the grant, excluding its signature. Each grant commits to one exact Resource function invocation made directly by the top-level RequestScript. All grants for a request are signed by its execution authority. A provider's receipt-signing key conveys no authority to issue grants, and providers cannot delegate their invocation permission. Local invocations use the same grants as remote invocations. Forwarding a direct invocation to its provider does not create another invocation or require a new grant.

Each provider verifies the bound on-chain authorization, the grant's execution-authority signature, exact target and parameter hashes, revision and quote terms, and deadlines before execution. The grant must reference the same request and authorization as its receipt, be issued no earlier than the request authorization, and expire no later than that authorization. The target revision must be active at grant issuance and at successful completion. Successful completion must occur no later than the grant's expiry. A grant expiring after completion does not invalidate a timely receipt submission while the request authorization and budget remain open. Settlement receives exactly one grant per claimed invocation. Grants contain hashes and payment metadata, not source, arguments, or outputs.

The node durably records invocation IDs before executing them. Re-delivery of a grant returns the recorded result or status and does not execute or charge again. If a crash leaves execution outcome unknown, the node returns an indeterminate status and does not automatically retry that invocation; the protocol cannot guarantee exactly-once effects at an external API. A deliberate second invocation, even with identical parameters and quote, needs a fresh invocation nonce and grant. Settlement permanently permits only one claim per `(requestId, invocationId)` across both acceptance and fallback paths.

A signature establishes who permitted a call; it does not prove that the call was required by the authorized program. The execution authority must issue grants only for Resource calls reached directly by the canonical top-level RequestScript. Arbitrators verify this relationship using the authorized source, inputs, invocation grant, and interpreter execution trace. A valid signature for an unrelated or nested call is grounds for a refund. The caller's total reserved budget remains the upper bound for all direct invocation claims.

## 7\. Request and settlement flow

```mermaid
sequenceDiagram
    participant C as Caller wallet and SDK
    participant N as Stellarium node
    participant P as Resource provider
    participant R as Selected relay
    participant S as Base settlement contracts
    C->>S: Deposit USDC; reserve budget bound to signed authorization
    C->>N: Signed request authorization and RequestScript source
    N->>P: Invoke paid function with request authorization and signed invocation grant
    P->>P: Execute successfully
    P-->>N: Response plus provider-signed receipt
    N-->>C: Response and receipt
    C->>P: Optional acceptance signature
    P->>R: Signed receipt and optional acceptance
    R->>S: Batch claims
    S->>S: Lock each claim for one-hour challenge period
    S-->>R: Release relay fee after challenge period
    S-->>P: Release provider net after challenge period
```

### 7.1 Invocation rules

1. The caller discovers a valid signed Resource manifest from a peer and verifies its current registry anchor. Its quotes remain active until that Resource's payment manifest is updated at node startup.
2. The wallet deposits USDC if needed, and the caller signs `RequestAuthorization`.
3. The wallet reserves the authorized request budget in `StellariumBalanceVault`, binding the authorization hash and fields. After confirmation, the caller submits the authorization with the RequestScript request to its execution authority.
4. Every paid local or remote invocation resolves the provider's quote and obtains an invocation grant before execution. The provider verifies the authorization and execution-authority-signed grant and records its invocation ID before executing. A local invocation serving an external caller follows the same payment process as a forwarded invocation.
5. A provider executes the function. A failure emits no execution receipt and consumes no request budget for that provider.
6. A successful provider returns the response and its signed receipt. The caller SDK normally signs acceptance after validating the response hash.
7. The provider or node forwards the receipt to the assigned relay. A relay batches compatible claims to reduce gas cost.

Only Resource calls made directly by the top-level RequestScript may consume the request budget. The script may sequence calls, use earlier results as later arguments, or invoke multiple providers; each direct invocation has its own grant and receipt. Resource functions cannot invoke other Resource functions through Stellarium, locally or remotely. Nodes reject attempts to invoke a Resource from within an executing Resource function, whether paid or free. Ordinary HTTP requests used to implement a Resource remain supported, but do not confer permission to make nested Stellarium invocations or generate additional claims. Forwarding a direct invocation to another node is transport for that same invocation, not a nested call. Contract settlement accepts claims in deterministic on-chain order until the budget is exhausted. The SDK should reserve sufficient headroom and present the total authorized budget clearly before the caller signs.

### 7.2 Claim validation

Before accepting a claim, `Settlement` automatically rejects:

- invalid caller, provider, or acceptance signatures;
- a provider, recipient, or receipt signer that does not match the referenced historical manifest revision;
- a missing invocation grant, a grant not signed by the request's execution authority, or mismatched invocation fields;
- a Resource, function, quote, or gross price that does not match both the invocation grant and its manifest revision;
- a manifest revision or quote that was not active at grant issuance or `executedAt`, an expired request authorization, a closed budget, or an expired submission window;
- an `executedAt` before authorization or grant issuance, after the invocation grant expiry, or later than the submission block timestamp;
- a duplicate `receiptId` or previously claimed `(requestId, invocationId)`, regardless of acceptance or fallback path;
- an authorization that does not match the open budget's bound authorization hash;
- a receipt whose requested amount exceeds the remaining request budget;
- a relay submission from an EOA other than the receipt's assigned relay;
- stale Chainlink oracle data older than 15 minutes.

Quote IDs and the bound request authorization are reusable across distinct invocations; their reuse alone never rejects a claim.

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
- the provider charged for another function or input;
- the provider falsified execution time, including backdating an execution to a retired quote or receipt-signing key;
- an invocation was unrelated to the authorized top-level request or was a prohibited nested call, even if its grant has a valid signature.

The court does not decide whether an arbitrary API result is substantively correct.

### 9.2 Evidence

On-chain records expose only hashes and payment metadata. The canonical request source and inputs, response, signed authorization, invocation grants, signed receipts, referenced manifest revisions, and execution trace are stored in encrypted peer-hosted evidence storage. The top-level interpreter trace binds direct Resource calls to invocation IDs and records dispatch and completion events. Nodes retain available signed dispatch acknowledgements and contemporaneous chain references as timing evidence. Provider timestamps and self-produced traces alone do not independently prove execution time or authorized control flow; the claimant must substantiate disputed timing and call relationships with the available corroborating evidence. If the panel cannot establish the disputed claim's validity, the locked claim is refunded to the caller.

Every active arbitrator publishes an encryption public key with its stake registration. After the randomness adapter selects a panel, each party releases the necessary encrypted evidence directly to each selected arbitrator.

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

All USDC transfers use safe ERC-20 wrappers. The implementation must use reentrancy guards, explicit state machines, and pull-payment withdrawal where applicable. Receipt IDs and `(requestId, invocationId)` claims are permanently deduplicated; request IDs and caller authorization nonces cannot open or bind multiple budgets. Quote IDs and manifest revision IDs are immutable references, reusable across distinct invocations, and cannot be reassigned to different terms.

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
3. Add a manifest cache, quote validation against current and historical registry commitments, peer relay-advertisement cache, and encrypted-evidence record exchange. At node startup, compare each Resource's payment manifest with its registered manifest; preserve quotes when unchanged, and anchor updated manifests with new quote IDs while retiring that Resource's previous quotes before accepting paid invocations.
4. Extend resource invocation so local and forwarded paid calls receive a request-scoped `PaymentExecutionContext` containing the bound request authorization, invocation grant, remaining budget estimate, trace, and selected relay. Persist invocation IDs and execution status to prevent re-execution after retries or restarts; on-chain remaining budget remains authoritative for concurrent claims.
5. Refuse paid execution without a valid execution-authority-signed invocation grant, active quote and manifest revision, registered payment keys, and sufficient reserved budget. Validate receipt keys against historical revisions at settlement.
6. Create an execution receipt only after `fn.exec` returns successfully. Do not create a receipt for exceptions, timeouts, or type-validation failures.
7. Carry the existing authorization and invocation grant when forwarding a direct call to its provider; forwarding must preserve its invocation ID. Replace the current string-built forwarding request with a structured and safely encoded request so parameter values and payment metadata cannot be injected into source text. Expose Resource invocation only to the top-level interpreter and reject nested invocation attempts from executing Resource functions, for paid and free Resources alike.
8. Return function output, execution receipt, provider evidence location, and quote data to the caller SDK.
9. Publish the receipt to the assigned relay and retain encrypted evidence until the challenge and appeal periods end.

### 14.2 TypeScript SDK

The SDK should provide:

- EIP-1193 injected-wallet and WalletConnect connection adapters; Coinbase Wallet works through either compatible connection path.
- Base Mainnet and Base Sepolia network configuration.
- USDC deposit, request-budget reservation, closing, and withdrawal flows.
- Resource, manifest revision, quote, and execution-authority-signed invocation grant verification before invocation.
- a prominent total request-budget approval screen showing the execution authority;
- automatic acceptance-receipt signing after response-hash validation;
- claim, challenge-window, dispute, and appeal status; and
- encrypted evidence creation and delivery to selected arbitrators.

The hosted wallet experience is a hosted user interface over external wallets. It never holds a private key or custody balance.

### 14.3 Solidity reference implementation

Implement each contract behind interfaces and deploy in this order on Base Sepolia:

1. USDC address configuration, `StellariumBalanceVault`, `ResourceRegistry`, and EIP-712 domain test harness.
2. `Settlement` with direct acceptance claims, then fallback claims, direct invocation grant validation, historical manifest-key validation, reusable-quote tests, invocation replay protection, and budget exhaustion tests.
3. `FeeOracle` with Chainlink freshness checks and Base fee-allocation tests.
4. `RelayRegistry`, deterministic off-chain selection test vectors, and receipt relay binding.
5. `ReputationRegistry` and sponsored-identity admission rules.
6. `RandomnessAdapter` and `ArbitrationCourt`, including all evidence-deadline and 3/5/7 appeal transitions.
7. `ProtocolTreasury`, Governor, timelock, launch multisig configuration, and governance-transfer checks.

## 15\. Rollout and verification

1. Publish protocol schemas, EIP-712 type hashes, canonicalization rules, deployment parameters, and signed test vectors.
2. Run end-to-end integration on Base Sepolia with independent caller, provider, relay, and arbitrator nodes.
3. Simulate failed calls, caller disconnects, duplicate claims, exhausted budgets, stale oracle data, relay front-running, evidence non-delivery, and every appeal outcome. Verify that quotes remain active regardless of age, unchanged node restarts preserve quotes, a Resource payment manifest update at node startup retires only that Resource's previous quotes, and receipts executed before retirement still settle within the existing deadlines.
4. Verify repeated purchases under one quote and authorization, duplicate receipt variants, cross-request grant substitution, grants signed by unauthorized providers, rejection of paid and free nested calls on local and remote paths, preservation of invocation IDs during forwarding, multiple direct calls sharing one budget, key rotation with pending receipts, falsified timestamps, disputed unrelated calls, and durable invocation recovery after restart.
5. Obtain an independent smart-contract audit and remediate findings before holding Mainnet USDC.
6. Deploy the same versioned contracts to Base Mainnet only after the Sepolia rollout and audit pass.
7. Publish public steward identities, multisig threshold, timelock addresses, contract addresses, oracle address, parameters, and the governance-transfer condition.

## 16\. Deferred design work

- Add a provider bond and slashing policy for unilateral fallback claims.
- Define appeal fees and potentially an economic bond for appeals.
- Evaluate proportional fee allocation against equal-per-receipt allocation.
- Add smart-account / ERC-1271 signatures.
- Consider subscriptions, quotas, usage pricing, and provider-defined price functions only in a future protocol version.
- Reassess sponsor accountability and anti-Sybil protection as reputation governance becomes active.
