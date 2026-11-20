# SIP-1 Contract Functions

Author: `Christopher Poulsen`
Status: `draft`

___


## Summary

Instead of contracts running plain old code top to bottom, they should have entrypoints, (defined as `pub fn`s)

Current Workflow:

```
contract MyContract@1(data: object) {
    return "Do thing!"
}
```

Refactored workflow:

```
contract com.example.MyContract@1 {
    pub fn doThing(data: string): string {
        return "Do thing!"
    }
}
```

State variables are declared at the top level of the contract, or at the top of a function's body.

```
contract com.example.MyContract@1 {
    var state myData = ""

    pub fn doThing(data: string): string {
        return "Do thing!"
    }
}
```

Contracts can also have private functions that can be used to organize the code and prevent duplication.

```
contract com.example.MyContract@1 {
    var state myData = ""

    fn sayThing(): string {
        return "Say thing!"
    }

    pub fn doThing(data: string): string {
        return sayThing()
    }
}
```

When calling a contract, you reference a function on it:
```
request MyRequest {
    const myContract: com.example.MyContract@1

    return myContract.doThing(data: "Hello World!")
}
```

This makes contracts more scalable, organized, and also allows contracts to do multiple things with state within one context, like CRUD applications.

## Contract body

A contract's body holds only these, in any order:

- `var state` variables,
- resource bindings, `const r: path.Name@version`,
- functions, `fn` and `pub fn`.

Any other statement outside a function is an error. The state variables and bindings are declared in source order every time the contract runs, before the function it runs.

The contract itself takes no parameters and has no return type. `contract X(params): T { ... }` and `payable contract X` are errors.

## Functions

```
[pub] [payable] fn name(parameter: type, ...)[: type] {
    ...
}
```

- **Entrypoints.** A `pub fn` is an entrypoint. A host runs a contract by naming one, and callers call it as `ref.name(...)`. A contract needs at least one.
- **Private functions.** A function without `pub` can only be called by the contract's own functions.
- **Parameters.** Every function declares its own. The parentheses are required, `()` when there are none.
- **Return types are optional.** A function without one returns nothing: it evaluates to `null`, and `return value` in it is an error. A bare `return` ends it.
- **Calls.** A function is called by its name with named arguments, `sayThing()` or `add(a: 1, b: 2)`. The call evaluates to what the function returns, not to a `{ value, error }` result, and a failure in it fails the run.
  - Any function can call any other, `pub` ones included.
  - Unknown functions, and missing or unknown arguments, are errors before the script runs.
- **No recursion.** A function can't call itself, directly or through other functions. Without loops or recursion, the most tokens a run can execute stays known before it runs.
- **Scope.** A function runs in its own scope beneath the contract's top level. It sees the contract's state variables and bindings, but not its caller's variables. Its parameters can't take the name of a top-level state variable or binding.
- **Reserved words.** `fn` and `pub` become reserved keywords.

## State

- State belongs to the contract and is keyed by name, so every function shares it.
- `var state` may be declared at the contract's top level, or at the top level of a function's body, where its scope can use the function's parameters:

```
contract com.example.Jobs@1 {
    var state total: int64 = 0

    pub fn pay(jobId: string, amount: int64): int64 {
        var state(jobId) paid: int64 = 0
        paid = paid + amount
        total = total + amount
        return paid
    }

    pub fn paidFor(jobId: string): int64 {
        var state(jobId) paid: int64 = 0
        return paid
    }
}
```

- Several functions may declare the same state variable. Every declaration must agree on its type and on whether it has a scope.
- Within one run, a variable declared again for the same scope, for example by a helper, is the same variable, not a second copy.
- A variable declared at the top level can't also be declared in a function.

## Payable functions

`payable` moves from the contract to its entrypoints, after `pub`:

```
contract com.example.Tip@1 {
    pub payable fn tip(note: string): int64 {
        return payment.amount
    }

    pub fn thanks(): string {
        return "Thank you!"
    }
}
```

- **Only a payable function takes money.** Money attached to a call to `thanks` is refused. Calls attach money between the function's name and its arguments: `tip.tip{amount: 1000000, payees: [...]}(note: "great")`.
- **Payable contracts.** A contract with at least one `pub payable fn` is a payable contract, as before. Every one of its functions reads `sender` and `now`, and every run of it is verified, whichever function runs.

## Event contracts

An event contract's handler is the reserved function `on`, which takes the fields of the event it reads:

```
contract com.example.OnOrderPlaced@1 consumes com.example.OrderPlaced@1 {
    fn on(orderId: string, total: decimal) {
        var state(orderId) seen: int32 = 0
        seen = seen + 1
    }
}
```

- `on` isn't `pub`, has no return type, and can't be called, since only events run it.
- Each of its parameters must be a field of the event, with the field's own type. It may leave fields out.
- An event contract has no `pub` functions, but may have private ones.
- A contract that consumes nothing can't declare a function called `on`.
- SIP-2's `pub fn render()` is a reserved entrypoint too, but it's `pub` because a viewer calls it.

## Stellarium

- **Resources.** A deployed contract's resource has a function for each `pub fn` of its latest version, marked `payable` when it takes money. Private functions aren't listed.
- **Versions.** A pinned older version keeps the functions it had, even ones its latest version dropped.
- **Vouchers.** A contract voucher signs the `functionName` it calls. The provider requires it to match the call.
- **Holds.** The operator holds what the named function can cost, rather than what the whole contract can.
- **Receipts.** A run's receipt records the function it ran, and the operator replays that function.

## Compatibility

This is a breaking change, released as RequestScript 2.0.0 together with stellarium-ts, the operator and the resource registry.

- **The old syntax no longer parses.** Contracts written for RequestScript 1 aren't migrated.
- **Nodes drop their old contracts.** A node upgrading to 2.0 drops its contracts and their state, so they must be deployed again as functions.
- **Older nodes aren't supported.** Nodes on older versions can't call contracts on 2.0 nodes.
