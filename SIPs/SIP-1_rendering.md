# SIP-1 Renderings

Author: `Christopher Poulsen`
Status: `draft`

___


## Summary

Renderings are full-stack applications built entirely with RequestScript code.

Example:

```
rendering com.example.CountUp@1 {
    var state("count") count = 0

    fn increaseCount() {
        count += 1
    }

    render {
        Column {
            Text("Count: ${count}")
                .color("blue")
            
            Button("Up", () => { increaseCount() })
        } 
    }
}
```

Renderings use many built-in objects and features like Text, TextFields, Buttons, etc. to build up powerful functionality.
Renderings can also call resources, contracts, and emit events like normal contracts can.

Renderings are accessed by the renderings' contract address on a node:

`https://192.168.0.1:1411/v1/r/com/example/CountUp/1`

Because Renderings are native RequestScript code, they could also easily translate to iOS and Android native apps, or MCP apps for AI Agents.

Renderings are a powerful way for AI agents to dynamically build full applications.

## Sign-In with Stellarium

Renderings can access user data (payment data, etc.) from Sign-In with Stellarium.
Using the built-in `signIn` object automatically prompts the user to sign-in and authorize the rendering to use specific user data.

For example, in the following rendering, Stellarium knows to request the user's `email` and `name` from the sign-in prompt.

```
rendering com.example.CountUp@1 {
    var state(signIn.email) count = 0

    fn increaseCount() {
        count += 1
    }

    render {
        Column {
            Text("Hello ${signIn.name}")
                .fontSize(20)
            Text("Count: ${count}")
                .color("blue")
            
            Button("Up", () => { increaseCount() })
        } 
    }
}
```

The sign-in prompt is automatically fetched and displayed when the user navigates to the rendering, and is saved between sessions.

## Protected

State functions the same as it does for contracts and is preserved across users. This is not ideal for sensitive fields like passwords. For this, renderings use the `protected` keyword.

The `protected` keyword keeps a variable value between renderings, but is not saved to state.

```
rendering com.example.Login@1 {
    // An example Auth Resource
    const auth: com.example.Auth

    protected var username = ""
    protected var password = ""

    var loggedIn = false

    fn checkLogin() {
        const authResult = auth.checkLogin(username, password)
        if (authResult.value != null) {
            loggedIn = authResult.value.loggedIn
        }
    }

    render {
        Column {
            if (loggedIn) {
                Text("Hello ${username}")
            } else {
                TextField("Enter Username", (usernameInput: string) => { username = usernameInput })
                TextField("Enter Password", (passwordInput: string) => { password = passwordInput })
            }
            
            Button("Log in", () => { checkLogin() })
        } 
    }
}
```
