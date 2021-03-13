---
layout: post
title:  "Let's apply Hexagonal Architecture!"
date:   2020-01-15 00:00:00
categories: architecture hexagonal
image-base: /assets/images/posts/2020-01-15-lets-apply-hexagonal-architecture
---

In the last article we [learnt about Hexagonal Architecture](https://sketchingdev.co.uk/blog/lets-learn-hexagonal-architecture.html) and how it is applied to our particular problem, but stopped short of creating a finished solution. In this post I hope to remedy this by creating a [solution using Hexagonal Architecture](https://github.com/SketchingDev/hexagonal-lambda), and to pay homage to the architecture’s renaissance in the Serverless world we will implement it using AWS Lambdas in NodeJS/TypeScript.

If you don’t know about Lambdas then don’t worry, the code we'll be writing shouldn't need you to understand them, although there are some great tutorials out there!

## Problem we want to solve

Let’s first remind ourselves of the problem from the last article…

![Outlines of customers being deleted]({{ page.image-base }}/problem.png)

At the energy company I work for we have a lot of customers in our systems and with the natural ebb and flow of any large organisation, we are regularly opening and closing accounts. Let us imagine then, that we’ve been tasked with writing a system that will automatically close these accounts. We do this by:

1. Removing all meters associated with the account
2. Telling the system managing the account to close it

A simple enough task until you learn that there are numerous sources wanting to close accounts, some want to put CSVs into S3 buckets, others favour HTTP endpoints and Customer Services have always wanted a UI. Similarly, downstream systems for managing accounts are also numerous.

## Applying our knowledge of Hexagonal Architecture

Before we start writing code let’s step back and look at the interactions we’ll want our application to have with the outside world:

1. Receiving the ID of the account to be closed
2. Telling the account’s management system to remove meters and flag the account as inactive
3. Monitoring our application

Each of these interactions can become a port, since if you remember a port is supposed to have a specific intention. The adaptors can then implement or invoke them, although not before we’ve fleshed out exactly what each port requires by implementing the core functionality….
![Hexagon with adaptors]({{ page.image-base }}/full-diagram.png)

_To spare you precious minutes of your life we’ll only write a single adaptor on either side of the hexagon. If you’re interested in these other adaptors thought you can view them in the [solution’s repository](https://github.com/SketchingDev/hexagonal-lambda)._

### Core functionality

![Core functionality]({{ page.image-base }}/core-functionality.png)

We’ll start by creating our core functionality which will close the account based on the ID that is provided. As the diagram above illustrates, this has three ports which will become interfaces.

1. `CloseAccount` — ‘For providing the ID of an account to close’
2. `AccountManager` — ‘For managing customer accounts’
3. `Instrumentation` — ‘For instrumentation’ 

Our core functionality implements the `CloseAccount` interface (which the left-hand adaptors will later invoke with the account ID) and will depend on implementations of the `AccountManager` and `Instrumentation`  interfaces (that the adaptors on the right will implement).

```typescript
export type CloseAccount = (accountId: string) => Promise<void>;

export const closeAccount = ({
  accountManager,
  instrumentation,
}: {
  accountManager: AccountManager;
  instrumentation: Instrumentation;
}): CloseAccount => async (accountId: string): Promise<void> => {
  const activeMeters = await accountManager.getActiveMeters(accountId);

  try {
    await Promise.all(activeMeters.map(m => accountManager.removeMeter(accountId, m)));
    await instrumentation.removedMeters(activeMeters);
  } catch (err) {
    throw new Error(`Failed to remove meters for account ${accountId}`);
  }

  await accountManager.closeAccount(accountId);
  await instrumentation.closedAccount(accountId);
};

```

Our core functionality is pretty self explanatory; all of its interactions with the outside world are performed through domain-specific ports, meaning it is completely agnostic of the technical infrastructure that it is going to interact with.

### Account Management Adaptor

We can now turn our attention to the technical infrastructure and how our application is going to interact with it. 

Let’s first start with an adaptor on the right, which interacts with the account management system. This will implement the port `AccountManager` (‘for managing customer accounts’) and allow for the management of our fictional Amazing Energy customers.

![Account Management Adaptor right of the hexagon]({{ page.image-base }}/right-adaptor.png)

Since we want to create a fully deployable solution, yet not burden ourselves with the complexity of setting up then calling a real management system, nor standing up an external mock API we can create a stub adaptor to simulate Amazing Energy customers for the purposes of this demonstration. 

Where we would have performed HTTP requests, our stub adaptor will instead just return the domain object — in the real world we would have mapped the HTTP response to this object.

```typescript
import { FuelType, Meter, Unit } from "../../../domain/models/Meter";
import { AccountManager } from "./AccountManager";

/**
 * Adaptor for managing AmazingEnergy customers
 */
export class StubAmazingEnergy implements AccountManager {

  private static readonly ELECTRICITY_METER: Readonly<Meter> = {
    id: “elec-id”,
    fuelType: FuelType.Electricity,
    lastKnownReading: {
      value: 123,
      unit: Unit.watts,
    },
  };

  private static readonly GAS_METER: Readonly<Meter> = {
    id: “gas-id”,
    fuelType: FuelType.Gas,
    lastKnownReading: {
      value: 456,
      unit: Unit.m3,
    },
  };

  public async closeAccount(): Promise<void> {
    return Promise.resolve();
  }

  public async getActiveMeters(): Promise<Array<Readonly<Meter>>> {
    return Promise.resolve([
      StubAmazingEnergy.ELECTRICITY_METER,
      StubAmazingEnergy.GAS_METER
    ]);
  }

  public removeMeter(): Promise<void> {
    return Promise.resolve();
  }
}
```

Although a stub we are still able to see the implementation of the interface that represents the port, and its three methods `closeAccount(...)`, `getActiveMeters()` and `removeMeter(...)` . These abstract away any complexity with the technical infrastructure from the core functionality.

### API Gateway Adaptor
The API Gateway Adaptor to the left of the hexagon is our next port of call (excuse the pun). Our HTTP adaptor will take a request from the API Gateway to close the account, extract the account ID and pass it to the port `CloseAccount` (‘for providing the ID of the account to close’).

![API Gateway Adaptor left of the hexagon]({{ page.image-base }}/left-adaptor.png)

The code for this adaptor isn’t complicated either, as all it is doing is trying to extract the ID, throwing an error if it can’t, else passing that ID to  `closeAccount` .

```typescript
export const apiGatewayAdapter = (next: CloseAccount): APIGatewayProxyHandler => async event => {
  const id = tryExtractId(event);
  if (!id) {
    return response(“Account not defined”, 500);
  }

  try {
    await next(id);
    return response(“Successfully closed account”);
  } catch (err) {
    console.error(err);
    return response(“Unknown error”, 500);
  }
};
```

Later in the article we’ll be configuring this as the handler (entry-point) for the Lambda, which is why the adaptor’s return type is `APIGatewayProxyHandler`.

### Wiring it together
Now that we have all the components; the core functionality with its well defined ports and the adaptors bridging the ports to the outside world, we can wire the application together! 

We’ll skip the [merits](https://twitter.com/jeremy_daly/status/1192090251046670337) of whether to wire everything together in one Lambda (one Lambda for **all** events) or in multiple Lambda’s (one Lambda **per** event) and just go with my favourite, the latter — one Lambda per event type. 

```typescript
// handlerHttp.ts

//...
import { CloseAccount, closeAccount } from "./app/domain/closeAccount";
import { apiGatewayAdapter } from "./app/instrastructure/driving/apiGatewayAdapter";


// Instantiate core functionality with its dependencies
const accountCloser: CloseAccount = closeAccount({
  instrumentation: new StubInstrumentation(), // Implements Instrumentation interface (port)
  accountManager: new StubAmazingEnergyClient(), // Implements AccountManager interface (port)
});

// Initialise the handler with the apiGatewayAdaptor which depends on the CloseAccount port
export const handler = apiGatewayAdapter(accountCloser);
```

The exported handler initialised above is what the Lambda invokes when it receives a HTTP event, as defined in the [deployment configuration](https://github.com/SketchingDev/hexagonal-lambda/blob/86df671d585752f73eb04e704e72676ca7555c20/serverless.yml#L24-L25).  This HTTP Request is processed by the `apiGatewayAdaptor`  and then the relevant information (Account ID)  is passed to the `accountCloser` .

## Conclusion
There it is… there is nothing special going on. We're just using interface segregation with the ports, and some dependency injection to wire together the adaptors with the core functionality. But by thinking in terms of ports and adaptors it has informed where the responsibilities lay and kept a separation between our business logic and the infrastructure it relies on.

Hopefully next time you are tasked with writing an application, or refactoring an old one you’ll take Hexagonal Architecture into consideration. You can use this [article’s repository](https://github.com/SketchingDev/hexagonal-lambda) as a guide.
