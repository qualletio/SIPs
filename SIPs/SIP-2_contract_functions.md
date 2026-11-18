# SIP-2 Contract Functions

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
        return "Do thing!
    }
}
```

State variables are declared within or outside of entrypoint functions.

```
contract com.example.MyContract@1 {
    var state myData = ""

    pub fn doThing(data: string): string {
        return "Do thing!
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