# Stellarium: a programmable, decentralized API network

Christopher Poulsen
[https://github.com/cjpoulsen](https://github.com/cjpoulsen)

**Draft, October 2026.** This paper describes both what exists today in the reference implementation ([stellarium-ts](https://github.com/qualletio/stellarium-ts), [requestscript-js](https://github.com/qualletio/requestscript-js)) and what is proposed. Each major mechanism is labeled:

- **Implemented**: present in the reference implementation.
- **Prototype**: present, but with known gaps listed in [Open problems](#11-open-problems).
- **Proposed**: design intent, to be specified in a SIP.

## Abstract

Web APIs have moved from SOAP to REST to GraphQL, and three problems have survived every generation: callers must handle partial failure across many endpoints, providers must build their own billing to get paid, and every integration needs its own credentials and access rules. Stellarium addresses all three with a network of independently operated nodes that share callable functions called Resources. Callers do not send HTTP requests to individual APIs. They send a short program written in RequestScript, a small, strictly typed language that always terminates. The program composes any number of Resources across the network and returns a single response. Errors from individual Resources come back as values the program can handle. Providers attach a price to each function, and the caller's node pays them from an on-chain escrow on Solana. Paying for a call is also the default form of authorization.

## 1. Motivation

Consider a client that needs data from four services. With REST it makes four requests, and it has to handle four sets of status codes, four retry policies, four authentication schemes, and four billing relationships. GraphQL moves the composition to the server, but only within one provider's schema: relationships are fixed when the schema is designed, and crossing provider boundaries means stitching schemas or federating gateways.

Stellarium starts from a different unit. The unit of work is not a request to an endpoint but a **program over a network of functions**:

1. **Composition belongs to the caller.** The caller decides how data from different providers fits together, per request, without a provider changing a schema.
2. **Failure is data.** A failing Resource call produces a value (`{ value: null, error: { message } }`) that the program can inspect, retry against another provider, or ignore. The caller sees one response for the whole program.
3. **Payment and access are part of the protocol.** A provider publishes a price next to the function signature. The network carries the payment authorization with the call, so the provider needs no billing system, sign-up flow, or key management.

## 2. System overview

```
  end user                caller node                         provider node
 ---------- API key ---> ------------  POST /v1/run/resource  -------------
 | client | -----------> | interpret | ---------------------> | Resource  |
 ---------- <----------- | RequestScript| <-------------------- | exec()    |
            one response ------------  { value, error }        -------------
                              |                                     |
                              | funds escrow                        | submits voucher
                              v                                     v
                        --------------------------------------------------
                        |  operator  ->  Solana escrow program           |
                        |  (pays provider 80%, protocol fee 20%)         |
                        --------------------------------------------------
```

| Role | Description |
| :--- | :--- |
| **Node** | A process running a Stellarium implementation (for example `stellarium-ts` embedded in a Fastify server). A node has an Ed25519 identity key, hosts zero or more Resources, keeps a list of peers and the Resources they host, and interprets RequestScript. |
| **Resource** | A named collection of typed functions implemented in the node's host language. It is addressed by a fully qualified name such as `com.example.Weather`. |
| **Caller node** | The node that receives a RequestScript program from its user, runs it, and pays for any priced Resource calls. |
| **Provider node** | The node that hosts a Resource and executes its functions. |
| **End user** | Whoever sends programs to a caller node. Stellarium does not define how end users authenticate to their own node or how a node bills them. Today a node checks a bearer API key. |
| **Operator** | The service that creates escrow accounts and settles payments on Solana. It is centralized in version 1 (see [Section 8](#8-trust-model)). |

## 3. RequestScript

**Status: Implemented** (`requestscript` 1.1.0).

RequestScript is a small, strictly typed scripting language with TypeScript and Kotlin flavored syntax. A program is a single `request` declaration:

```
request HelloWorld {
    const message = "Hello World!"
    return message
}
```

### 3.1 Total by design

RequestScript has variables, objects, lists, arithmetic, string interpolation, and `if`/`else`. It deliberately has **no loops, no recursion, and no user-defined functions**. This is a deliberate choice, and it gives three guarantees that a general-purpose language cannot:

1. **Every request terminates.** A node can accept programs from untrusted callers without a step counter or gas metering.
2. **The worst-case cost is known before execution.** A program can call a priced function at most once per call site, so its maximum cost is the sum of the prices of every call expression in the source. A caller node can show or enforce this bound before running anything (see [Section 6.3](#63-cost-bounds)).
3. **Programs are easy to analyze.** Every Resource a program can touch is declared with `const name: path.Name`, so a node can resolve, price, and authorize all of them up front.

Workloads that need iteration belong inside a Resource, where the provider controls and prices the work.

### 3.2 Type system

| Type | Values |
| :--- | :--- |
| `int32`, `int64` | Signed integers, range checked, never wrap |
| `decimal`, `decimal(p, s)` | Exact fixed-point decimals (`0.1 + 0.2 == 0.3`) |
| `boolean` | `true`, `false` |
| `string` | Text |
| `object` | String-keyed maps of any values |
| `[]T` | Lists of `T` |

`null` belongs to every type. There is no truthiness: `if (1)` is an error. Numbers are promoted from `int32` to `int64` to `decimal` where needed, and no other implicit conversions happen. Exact decimals matter for an API network that handles prices and quantities.

### 3.3 Calling Resources

A program binds a Resource by name and calls its functions with named arguments:

```
request GetTemperature {
    const weather: com.example.Weather

    const result = weather.temperature(city: "Oslo")

    if (result.error != null) {
        return { ok: false, reason: result.error.message }
    }

    return { ok: true, celsius: result.value }
}
```

A call never evaluates to the bare return value. It evaluates to a result object with exactly one of `value` or `error` set. A Resource that throws, times out, or cannot be reached produces an `error`, and the program continues. Only a defect in the program itself (a type mismatch, an unknown variable, division by zero) aborts it.

### 3.4 Composition across providers

Here a program combines two independent providers, and one of them is optional:

```
request CityReport {
    const weather: com.example.Weather
    const air: org.example.AirQuality

    const temperature = weather.temperature(city: "Oslo")
    if (temperature.error != null) {
        return { error: temperature.error.message }
    }

    const quality = air.index(city: "Oslo")
    var aqi = null
    if (quality.error == null) {
        aqi = quality.value
    }

    return {
        city: "Oslo",
        celsius: temperature.value,
        airQualityIndex: aqi,
    }
}
```

And here a program falls back across providers, which takes custom client code in a REST integration:

```
request ResilientTemperature {
    const primary: com.example.Weather
    const backup: net.example.Forecasts

    var reading = primary.temperature(city: "Oslo")
    if (reading.error != null) {
        reading = backup.currentTemperature(city: "Oslo")
    }

    return reading
}
```

The response structure is set by the caller in each program, not by a provider's schema. Nested shapes are just object literals:

```
request Body {
    const anatomy: com.example.Anatomy

    const head = anatomy.part(name: "head")
    const shoulders = anatomy.part(name: "shoulders")
    const knees = anatomy.part(name: "knees")
    const toes = anatomy.part(name: "toes")

    return {
        head: head.value,
        torso: { shoulders: shoulders.value },
        legs: { knees: knees.value, toes: toes.value },
    }
}
```

### 3.5 What "one status code" means

The caller of `POST /v1/run` gets one HTTP response per program. It is `200` with `{ "returnValue": ... }` when the program completes, even if individual Resource calls failed, because those failures are values inside `returnValue`. It is an error status only when the program itself is invalid or the node fails. Partial failure is still possible. Stellarium does not make it disappear; it moves the handling into the program, where the caller writes it once, in one language, for every provider.

### 3.6 Execution model and latency

**Status: Implemented** (sequential). The reference interpreter evaluates calls in source order and waits for each before continuing. The latency of a program is therefore the sum of its calls, plus one network hop from the caller node to each provider. Because programs have no loops and their dependencies between calls are visible in the source, independent calls (such as `temperature` and `quality` above) could be run concurrently without changing what the program means. **Proposed:** concurrent evaluation of independent calls.

### 3.7 Contracts

**Status: Implemented in requestscript-js, not yet on the Stellarium network.** A `contract` is a saved, parameterized, immutably versioned program (`contract shop.orders.LineTotal@2(item: object) { ... }`). A Stellarium node today accepts only `request` programs. **Proposed:** a node may publish a contract as a composite Resource, with its own name and optionally its own price, so programs can be shared and reused across the network.

## 4. Resources

**Status: Implemented.**

A Resource is defined in the host language and registered with the node:

```typescript
const weather: StellariumResource = {
  metadata: {},
  path: "com.example",
  name: "Weather",
  functions: [
    {
      name: "temperature",
      parameters: [{ name: "city", type: "string" }],
      returnType: "int32",
      paymentManifest: { costLamports: 5000, recipientAddress: "<Solana address>" },
      exec: async (args) => lookupTemperature(args),
    },
  ],
};
```

The function's signature (name, parameter types, return type) and its optional payment manifest are public. The implementation never leaves the provider node. The interpreter checks argument types before the call is sent and checks the return value against `returnType` when it comes back, so a provider cannot silently return data of the wrong shape.

## 5. Networking and discovery

**Status: Prototype.**

### 5.1 Node identity

Each node has an Ed25519 key pair. The node signs what it publishes to peers (its own address and its Resource list) with this key, and the signatures expire after 10 minutes. Ed25519 is also Solana's signature scheme, so the same public key is the node's Solana address. That is how the operator links a payment voucher to an escrow account without a separate wallet binding (see [Section 6](#6-monetization)).

### 5.2 Joining the network

A node joins by naming one existing node, its *starting peer*. Once it is listening, the joining node:

1. Fetches `GET /v1/peers` from the starting peer and stores the list if it has no peers yet.
2. Fetches `GET /v1/resources` and stores each Resource along with the base URL of the node that hosts it.
3. Announces itself to every known peer with a signed `PUT /v1/peers`.
4. Announces its own Resources to every known peer with a signed `PUT /v1/all-resources`.

### 5.3 Routing

When a program binds `com.example.Weather`, the node looks it up locally. If the Resource is registered in the same process, the node calls `exec` directly. Otherwise it sends `POST {baseUrl}/v1/run/resource` to the hosting node with the function name, the typed arguments, and a payment voucher when the function has a price. Each call takes exactly one hop: providers do not forward calls, and Resource functions cannot call other Resources through Stellarium.

### 5.4 Scaling limits

Every node currently stores the full set of peers and Resources, and that set is updated only when the node joins or a peer announces itself. This works for a small network and will not work for a large one. **Proposed:** periodic signed gossip with versioned Resource records, replacement of stale records, and eventually a structured lookup (such as a DHT keyed on the fully qualified Resource name) so a node only needs to know the Resources it calls.

## 6. Monetization

**Status: Prototype** (Solana localnet). The detailed protocol is [SIP-1](SIPs/SIP-1_monetization.md).

### 6.1 Design choices

- **The asset is SOL, priced in lamports** (1 SOL = 10^9 lamports). SOL is native to Solana, so the escrow program needs no token accounts or mint handling. The node's Ed25519 identity is already a Solana address. Lamports are small enough to price very cheap calls. The cost of this choice is price volatility: a price fixed in lamports changes in fiat terms as SOL's price moves. Providers can re-price when their node restarts. A stablecoin option is listed under [Open problems](#11-open-problems).
- **The caller node pays.** Escrow accounts belong to node identities, not end users. A node works like an account holder or an ISP: it funds its escrow and decides for itself whether and how to charge the end users who send it programs. This keeps wallets out of end-user clients and lets a node offer free, subscription, or resold access on top of a usage-priced network.
- **Prices are fixed per function.** A function has one price per call, regardless of its inputs. Pricing that depends on inputs, metered pricing, and free quotas are out of scope for version 1.

### 6.2 Payment flow

```
caller node                 provider node                operator              Solana
    |  1. create + fund escrow -------------------------------> | ----------------> |
    |                                                                                |
    |  2. read price from GET /v1/resources                                          |
    |  3. sign voucher                                                               |
    |------ POST /run/resource + voucher ---->|                                      |
    |                                         | 4. verify voucher, execute           |
    |<------------- { value, error } ---------|                                      |
    |                                         | 5. POST /v1/payment (voucher) ------>|
    |                                         |              6. verify, dedupe ----->|
    |                                         |              withdraw: 80% provider  |
    |                                         |                        20% protocol  |
```

1. **Escrow.** The caller node's operator asks the operator service to create an escrow account. Its address is a program-derived address from the escrow program, the operator authority, and the node's public key. The node then funds it with SOL.
2. **Price discovery.** A priced function advertises `{ costLamports, recipientAddress }` in `GET /v1/resources`.
3. **Voucher.** Before calling a priced function, the caller node checks the price against its spending limit and signs a payment voucher with its identity key. The voucher contains a request ID, an invocation ID, the amount, the recipient, an expiry two minutes ahead, the Resource path and name, the call's arguments, and the signer's public key. Binding the arguments into the signature means a voucher authorizes one specific call and nothing else.
4. **Execution.** The provider verifies the voucher's signature and executes the function.
5. **Claim.** If execution succeeds, the provider submits the voucher to the operator. A function that throws produces no claim.
6. **Settlement.** The operator verifies the signature and expiry. It rejects any `(requestId, invocationId)` it has already settled, then has the escrow program withdraw the amount: 80% to the provider's recipient and 20% to the protocol fee wallet, both enforced by the program. The operator records the receipt and transaction signature.

### 6.3 Cost bounds

**Status: Proposed.** Today a caller node enforces a fixed limit on each call. Because RequestScript has no loops (Section 3.1), a node can do better: before execution, it can add up the prices of every priced call site in the program and reject the program, or ask the end user to confirm, if that maximum exceeds a budget. Version 1 should make this per-request budget the primary spending control and bind it into every voucher of the request with a shared request ID.

### 6.4 Economics

| Item | Who bears it |
| :--- | :--- |
| Function price | Caller node, from escrow |
| Protocol fee (20% of price) | Taken from the price, so the provider receives 80% |
| Solana transaction fees and escrow rent | Operator, in version 1 |

Settling each call in its own transaction is simple but expensive at the bottom of the price range. Solana's base fee is 5,000 lamports per signature, so a function priced at or below that costs the network more to settle than it pays. **Proposed:** batched settlement, where the operator (and later the validator network) settles many vouchers for the same provider in one transaction, which makes sub-cent prices viable.

The 20% protocol fee funds the operator during version 1. Its rate, recipient, and use should be set by governance through the SIP process rather than compiled into the program, as they are today.

## 7. Permissioning

**Status: Proposed.**

### 7.1 Payment is permission

A function with a price is open to any caller that presents a valid, funded voucher. The voucher already authenticates the caller node (it is signed with the node's identity key) and authorizes the payment, so for most paid APIs no separate credential is needed. Providers do not issue API keys, run sign-up flows, or store customer accounts.

For this rule to hold, a provider node **must reject** any call to a priced function that arrives without a valid voucher whose amount and recipient match the provider's own manifest.

### 7.2 Access manifests

A provider can also attach an access manifest to a Resource or function to restrict who may call it, whether or not it has a price:

```typescript
accessManifest: {
  allow?: string[];   // caller node public keys; if present, only these may call
  deny?: string[];    // caller node public keys that may never call
}
```

The provider proves who the caller is from a signature: the voucher for priced calls, or a request signature over the same invocation fields for free calls. Access manifests are enforced only by the provider node; they are not secret and not enforced by the network.

### 7.3 End users

How end users authenticate to their own node, and what they are allowed to do there, is the node operator's business. The reference implementation checks a single bearer API key. Stellarium identifies nodes, not people.

## 8. Trust model

Stellarium version 1 is decentralized in discovery and execution, but **not** in settlement. This table states what each party can do and what stops them.

| Party | Can | Mitigation (version 1) | Mitigation (proposed) |
| :--- | :--- | :--- | :--- |
| Provider | Claim a voucher for a call it did not run correctly, or return bad data | Vouchers are bound to one call's arguments and expire after 2 minutes; the return type is checked | Caller acceptance receipts, a challenge window, and reputation (SIP-1, future work) |
| Provider | Overcharge | The caller signs the exact amount and checks it against its limit | Per-request budgets (Section 6.3) |
| Caller node | Receive a result and refuse to pay | Cannot: the voucher is signed before the call | N/A |
| Caller node | Replay a voucher | The operator deduplicates `(requestId, invocationId)` | On-chain replay protection |
| Peer | Advertise a false `baseUrl` for someone else's Resource and intercept its calls | **None today** | Resource records signed by the owning key (Section 9) |
| Operator | Withdraw escrowed funds to any address, or upgrade the escrow program | **None: the operator is trusted** | Program-level voucher verification, then a validator network |
| Operator | Go offline, locking caller funds | **None: callers cannot withdraw their own balance** | A refund instruction signed by the caller node (SIP-1, Section 10) |

The last row deserves emphasis. In version 1 the escrow program checks the operator's signature, not the caller's voucher. Callers are trusting the operator with the funds in their escrow. Removing that trust comes in two steps:

1. Verify the caller's Ed25519 voucher signature in the escrow program itself, using Solana's Ed25519 signature-verification program, so the operator can only move funds a caller actually authorized.
2. Replace the single operator with a validator network that submits settlements, so no single party controls when or whether providers are paid.

## 9. Naming

**Status: Proposed.** Resource names are reverse-domain paths (`com.example.Weather`). Nothing stops two nodes from claiming the same name, and today the most recent announcement wins. This is the most important open problem in discovery. Candidate designs:

- **Key-bound names.** A Resource record is signed by the key that first published it, and later updates must be signed by the same key. Simple, but first come, first served invites squatting.
- **Domain-verified names.** `com.example.*` is accepted only from a key published at a well-known location on `example.com` (a DNS TXT record or an HTTPS file). This reuses the ownership the web already has.
- **On-chain registry.** Names are registered on Solana. This is the most authoritative option, and the most expensive and least private.

Stellarium adopts a hybrid of the first two. Every name is key-bound: the first signed record for a name fixes its owning key, and updates must be signed by that key. A name under a real domain may additionally be domain-verified, and a domain-verified record takes precedence over a key-bound record for the same name. This lets the domain owner reclaim a squatted name. Names outside any real domain stay first come, first served.

## 10. Related work

| System | Similarity | Difference |
| :--- | :--- | :--- |
| REST / OpenAPI | Typed, documented functions over HTTP | One provider per request; the client composes and handles failure |
| GraphQL / federation | The caller shapes the response | The schema and the relationships between types are fixed by providers |
| API marketplaces (for example RapidAPI) | One account and one bill across many APIs | Centralized catalogue, billing, and proxy |
| HTTP 402 payment schemes (for example x402) | Payment travels with the request | Payment per HTTP request rather than across a composed program; no shared discovery |
| Oracle networks (for example Chainlink Functions) | Off-chain computation paid for on-chain | Built to deliver results to smart contracts, not to API consumers |

Stellarium combines caller-side composition, failure as data, and protocol-level payment in a single request, and it is the only one of these that does so over an open, peer-hosted directory.

## 11. Open problems

These are known gaps between this paper and the reference implementation, and design questions without an accepted answer. Each one is a candidate SIP.

1. **Priced calls without vouchers.** Provider nodes do not yet enforce Section 7.1.
2. **Resource hijacking.** Resource announcements are not signed by the Resource's owner, so a peer can redirect calls (Sections 8 and 9).
3. **Operator trust.** The escrow program trusts the operator rather than verifying vouchers, and a caller node cannot withdraw its own unspent balance (Section 8).
4. **Provider honesty.** "Pay only for successful calls" currently depends on the provider submitting only vouchers for calls that succeeded.
5. **Settlement cost.** One transaction per call is uneconomic for cheap functions (Section 6.4).
6. **Price stability.** SOL-denominated prices drift in fiat terms. Options: an SPL stablecoin escrow alongside SOL, or prices quoted in USD and converted at settlement using an oracle.
7. **Discovery at scale.** Full replication of the peer and Resource lists (Section 5.4).
8. **Concurrency.** Sequential execution makes a program's latency the sum of its calls (Section 3.6).
9. **Privacy.** Providers see call arguments in plain text, and the operator stores them with each receipt. Storing a hash of the arguments instead would remove that exposure from settlement.

## 12. Roadmap

| Phase | Scope |
| :--- | :--- |
| **0: Prototype** (today) | Peer bootstrap, RequestScript execution across nodes, SOL vouchers, a centralized operator, an escrow program on localnet |
| **1: Hardening** | Required vouchers for priced calls, signed Resource ownership, per-request budgets, voucher verification in the escrow program, devnet deployment |
| **2: Scale** | Batched settlement, gossip-based discovery, concurrent call evaluation, contracts as Resources |
| **3: Decentralization** | A validator network replaces the operator; protocol fee parameters are set by governance |

## 13. Governance

Changes to the protocol, the language, and the fee parameters are proposed as Stellarium Improvement Proposals in [this repository](https://github.com/qualletio/SIPs) and discussed in the open. A governance team is forming in the [Stellarium Discord](https://discord.gg/WWTWKmYWv6).
