# Stellarium: a next-generation programmable, decentralized API Platform

Christopher Poulsen  
[https\://github.com/cjpoulsen](https://github.com/cjpoulsen)

    Over the decades of software development, there have been many web-based API formats, frameworks, and guidelines. From SOAP, to REST, to GraphQL, there is always an evolution in how web-based APIs are composed. Three challenges have persisted across these frameworks: handling of status codes, monetization, and permissioning (or authentication/authorization). Stellarium seeks to improve on these formats by solving these challenges.

## Decentralization

Decentralization is the distribution of data, functionality, and decisions across multiple nodes, instead of controlled by a central authority. Decentralization is a key factor in Stellarium’s design, as it allows users to control their own requests and their APIs, while integrating with a wider network. It also means control of Stellarium is not limited to one party.

## Programming

    Rather than a notation format like JSON, or a query language like GraphQL, Stellarium requests are made with RequestScript–a near turing-complete language influenced by Kotlin and Typescript.

```
request MyRequest {
    const message \= “Hello World\!”

    return message
}
```

RequestScript allows maximum flexibility within requests to the server.

```
// Example 1: data validation
request MyRequestWithValidation {
    const name \= “Foo”

    if (name.length() \> 3\) {
        return {}
    } else {
        return {
            name: name,
        }
    }
}
```

```
// Example 2: combining data from multiple sources
request MyCombinedRequest {
    const resourceA \= [path.to](http://path.to).ResourceA
    const resourceB \= [path.to](http://path.to).ResourceB

    return {
        resourceAData: resourceA.getData()
        resourceBData: resourceB.getData().first
    }
}
```

In example 2, the user calls two Resources. Resources in RequestScript are external services defined in the host language (TypeScript, Go, etc.) which can be invoked from within the request. In a REST API, the user would have to make two separate calls and combine the data themselves, while managing retries and http status codes. In GraphQL, the relationship is locked-in at the schema level, and could require a substantial effort to change it. In RequestScript, the user is free to mix and match the data how they see fit, while only worrying about one status code: the request to the Stellarium server. Now imagine a complex, production-grade schema with many relationships. RequestScript again simplifies this:

```
request MyCombinedRequest {
    const headResource: path.to.Head
    const shouldersResource: [path.to](http://path.to).Shoulders
    const kneesResource: [path.to](http://path.to).Knees
    const toesResource: path.to.Toes

    return {
        head: head,
        torso: {
            shoulders: shoulders,
        },
        legs: {
            knees: knees,
            toes: toes,
        },
    }
}
```

This still only has the user managing one status code, one retry, and one request to a server, while keeping the schema flexible and decoupled.

## Monetization

Many developers want to monetize their APIs. They end up creating complex integrations with providers such as Stripe. For usage-based integrations, this requires introducing logic to track usage directly in the application code. Since Stellarium allows users to integrate with any API on the platform, it also knows when to charge users for their activity, and present payment back to the called API.  
As Stellarium is a decentralized platform, it makes sense that payment would take the form of cryptocurrency, namely USDC stablecoin. USDC is chosen for its relation to fiat US dollars, making its value easy to understand. USDC was also chosen for fast settlement times and low transaction costs, as individual resources called in a single request need to each be paid out.

## Permissioning

Permissioning (or authentication and authorization) is a tricky subject in web-based APIs. Stellarium abstracts these concepts away and allows users to write permission manifests along with their Resource definitions to give permissioned access to Resources.
