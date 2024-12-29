# Mental Poker Part 7: Primitives

For an overview on Mental Poker, see **[Mental Poker Part 0: An
Overview](https://vladris.com/blog/2023/02/18/mental-poker-part-0-an-overview.html)**.
Other articles in this
series **[here](https://vladris.com/writings/index.html#mental-poker)**.
In the previous **[post in the
series](https://vladris.com/blog/2024/04/07/mental-poker-part-6-shuffling-implementation.html)** we
saw how to implement shuffling on top of our primitives.

It this post, we’ll look at a few other primitives useful for implementing a
game on top of this toolkit.

## Creating a transport

We talked about [Fluid Framework](https://fluidframework.com/) in previous
posts. In [part
2](https://vladris.com/blog/2023/06/04/mental-poker-part-2-fluid-ledger.html),
we discussed the Fluid ledger, a distributed data structure which forms the
basis of our game message exchange. In [part
3](https://vladris.com/blog/2023/11/28/mental-poker-part-3-transport.html), we
talked about our `ITransport` interface and how we can implement it given a
ledger. We have’t covered how to get a ledger. 

Let’s go back down the stack, all the way to Fluid Framework. Fluid Framework
expects clients to agree on the basic layout of the distributed data structures
they’re working with. These data structures are packaged in a *container*. Note
this container has nothing to do with Docker containers, it’s simply a
definition for a set of data structures.

We’ll look at a simple implementation of joining a Fluid session and using a
container that includes only a ledger. We won’t even try to connect to an
instance of the Azure Fluid Relay service, rather we’ll use a local server.
Instructions for connecting to a service hosted in Azure are
[here](https://learn.microsoft.com/en-us/azure/azure-fluid-relay/how-tos/connect-fluid-azure-service).
For our local server, we need a stub user and an `AzureLocalConnectionConfig`
including an `InsecureTokenProvider` - this is all plumbing to connect to a
local instance of the Fluid Relay service:

```typescript
const user = {
    id: "userId",
    name: "userName",
};

const localConnectionConfig: AzureLocalConnectionConfig = {
    type: "local",
    tokenProvider: new InsecureTokenProvider("", user),
    endpoint: "http://localhost:7070",
};
```

With this connection config, we can now define a simple container containing a `Ledger`:

```typescript
export async function getLedger<T>(): Promise<ITransport<T>> {
    const client = new AzureClient({ connection: localConnectionConfig });

    const containerSchema = {
        initialObjects: { myLedger: Ledger },
    };

    let container: IFluidContainer;
    const containerId = window.location.hash.substring(1);
    if (containerId) {
        ({ container } = await client.getContainer(
            containerId,
            containerSchema
        ));
    } else {
        ({ container } = await client.createContainer(containerSchema));
        const id = await container.attach();
        window.location.hash = id;
    }

    const ledger = container.initialObjects.myLedger as Ledger<string>;

    return makeFluidClient(ledger);
}
```

We check the browser window’s URL: if it ends with a GUID, we load the
container; if not, we create a new container and add its GUID to the browser
window’s URL. This makes it easy to connect two local clients to the same
session:

* We launch our web app and the first client will create a container and get a
  GUID.
* We then copy/paste the URL into a separate tab and the second client will
  connect to the same session and load the container identified by the GUID.

The code above can be found in the
[`demos/transport`](https://github.com/vladris/mental-poker-toolkit/tree/main/demos/transport)
package. This is used by the other demo apps. Note you need to run the Fluid
Framework local service: `npx @fluidframework/azure-local-service@latest`.

We now have a simple abstraction, `getLedger()`, that wraps all the Fluid
Framework-specifics and gives us back an `ITransport` interface (implemented as
a `FluidTransport`).

## Upgrading the transport

We are building a turn-based, cryptographically secure game, so the first step
is to ensure our channel is secure and clients can’t spoof each other.

In [part
3](https://vladris.com/blog/2023/11/28/mental-poker-part-3-transport.html) we
looked at the `ITransport` interface, the `FluidTransport` implementation which
leverages the Fluid protocol for communication, and the `SignedTransport`
implementation which wraps the `FluidTransport` and enhances it with signature
verification.

> **Recap of signing**: in cryptography, we do signing using a public/private
key pair. These are both generated from a shared seed. Alice can sign a message
using her private key and anyone that has the public key, including Bob, can
verify that the signature is indeed Alice’s.
>
> So given a public/private key pair $<K_{private}, K_{public}>$ and some payload
$P$, singing is a function that produces a signature given the payload and
private key $sign(P, K_{private}) -> signature$. Signature verification is a
function that takes a payload, signature, and public key and tells us whether
the signature was indeed produced by the corresponding private key $verify(P,
signature, K_{public}) -> true/false$.

The neat thing about public/private key cryptography is that the public key,
which is required for validation, is not a secret - only the private key is.
Nobody can spoof a signature unless they have the private key (which isn’t
shared), but everyone with the public key can verify that the signature comes
from the private key owner.

So if we start with a `FluidTransport`, we need our clients to exchange public
keys. Each client generates a public/private key pair, and posts its client ID
and public key. We use these to populate the key store.

We can implement this on top of the state machine we saw in [part
5](https://vladris.com/blog/2024/03/22/mental-poker-part-5-state-machine.html).
First, we define our action and context. As a reminder, the action is what we
send over the wire and expect to receive. The context is an object we make
available to the code we run whenever an action appears over the transport.

```typescript
type KeyExchangeAction = {
    clientId: ClientId;
    type: "KeyExchange";
    publicKey: Key;
};

type CryptoContext = {
    clientId: ClientId;
    me: PublicPrivateKeyPair;
    keyStore: KeyStore;
};
```

In our case our action contains the `ClientId`, the `type` (which is
`KeyExchange`), and a public key. Each client is expected to post this over the
transport. The context contains our `ClientId` (so we can tell whether the
message came from us or someone else), our public/private key pair, and the
`KeyStore` in which we put all `ClientId`-to-`Key` mappings.

A helper function to create the `CryptoContext`:

```typescript
async function makeCryptoContext(clientId: ClientId): Promise<CryptoContext> {
    return {
        clientId,
        me: await Signing.generatePublicPrivateKeyPair(),
        keyStore: new Map<ClientId, Key>(),
    };
}
```

This leverages the cryptography primitives in our toolkit to generate a
public/private key pair.

Our sequence to be executed by the state machine is:

```typescript
function makeKeyExchangeSequence(players: number) {
    return sm.sequence([
        sm.local(
            async (
                actionQueue: IQueue<KeyExchangeAction>,
                context: CryptoContext
            ) => {
                await actionQueue.enqueue({
                    type: "KeyExchange",
                    clientId: context.clientId,
                    publicKey: context.me.publicKey,
                });
            }
        ),
        sm.repeat(
            sm.transition(
                (action: KeyExchangeAction, context: CryptoContext) => {
                    if (action.type !== "KeyExchange") {
                        throw new Error("Invalid action type");
                    }

                    if (action.clientId === undefined) {
                        throw new Error("Expected client ID");
                    }

                    if (context.keyStore.has(action.clientId)) {
                        throw new Error(
                            "Same client posted key multiple times"
                        );
                    }

                    context.keyStore.set(action.clientId, action.publicKey);
                }
            ),
            players
        ),
    ]);
}
```

Refer to [part
5](https://vladris.com/blog/2024/03/22/mental-poker-part-5-state-machine.html)
for the state machine details and a more in-depth explanation of local
actions/transitions etc. Our sequence starts with a local action, meaning
originating from our client: we post our client ID and public key. Then, for the
given number of `players` we expect in the session, we repeatedly expect an
incoming action of type `KeyExchangeAction`.

In other words, our protocol require each client to start by posting their
public key, and each client should expect as many such key postings as clients
in the game.

We handle some error cases:

* If the incoming action type is not a `KeyExchangeAction`, one of the clients
  didn’t respect the protocol, so we bail.
* If we don’t have a client ID, we also bail.
* Same if we already saw a key for this client ID - this means either a
  malicious client is trying to pretend to be another client ID, or a bug in how
  the protocol was implemented. Regardless, we have to bail.

If we didn’t hit any of these issues, then we store the client ID and key in the
`KeyStore` instance. Once the state machine executes this sequence, each client has enough
information to create a `SignedTransport`. Here is a helper function to perform
the whole key exchange:

```typescript
async function keyExchange(
    players: number,
    clientId: ClientId,
    actionQueue: IQueue<BaseAction>
) {
    const context = await makeCryptoContext(clientId);

    const keyExchangeSequence = makeKeyExchangeSequence(players);

    await sm.run(keyExchangeSequence, actionQueue, context);

    return [context.me, context.keyStore] as const;
}
```

This function takes as input the expected number of players, the ID of this
client, and an action queue (as discussed in [part
4](https://vladris.com/blog/2024/03/16/mental-poker-part-4-actions-and-async-queue.html)).
The implementation is straight-forward:

1. We create a `context`.
2. We generate a key exchange sequence by calling the function we just saw.
3. We use our state machine to run the sequence.
4. We return our private key and the `KeyStore` (the key store contains only
   public keys).

And here is a helper function that upgrades a transport to a signed one:

```typescript
export async function upgradeTransport<T extends BaseAction>(
    players: number,
    clientId: ClientId,
    transport: ITransport<T>
): Promise<IQueue<T>> {
    const [keyPair, keyStore] = await keyExchange(
        players,
        clientId,
        new ActionQueue(
            transport as unknown as ITransport<BaseAction>,
            true
        )
    );

    return new ActionQueue(
        new SignedTransport(
            transport,
            { clientId, privateKey: keyPair.privateKey },
            keyStore,
            new SignatureProvider()
        )
    );
}
```

This function takes the number of players, our client ID, and an `ITransport`
which doesn’t support signature verification. It executes the key exchange, then
creates a `SignedTransport` since it now has all the pieces needed for that.
This function goes a step further, and also initializes an async queue on top of
the singed transport.

A game that uses the toolkit can go from start to a queue over a signed
transport in 3 steps:

```typescript
const ledger = await getLedger<Action>();

const id = randomClientId();

const queue = await upgradeTransport(2, id, ledger);
```

In this example, we call `getLedger()`, which we discussed in the first part of
this post, we generate a unique client ID, then we call `upgradeTransport()`.
With these 3 lines of code, we get an `ActionQueue` over a `SignedTransport`.

## Establishing turn order and shared large prime

The last primitive we’ll look at in this post is another key component of Mental
Poker: having clients agree who goes first, and agree on a shared large prime
(this shared prime is used to generate SRA keys, as discussed in [part
1](https://vladris.com/blog/2023/03/14/mental-poker-part-1-cryptography.html)).

These can be separate steps but we can combine them to be more efficient. To
establish turn order, we can leverage the ledger distributed data structure
which guarantees all clients get all ops in the same sequence: each client posts
*something*, then we simply use the order in which clients see these posts as
the turn order.

Here’s a sketch of the state machine for this:

```typescript
type EstablishTurnOrderAction = BaseAction;

type EstablishTurnOrderContext = {
    clientId: ClientId;
    turnOrder: ClientId[];
};

function makeEstablishTurnOrderSequence(players: number) {
    return sm.sequence([
        sm.local(async (actionQueue: IQueue<EstablishTurnOrderAction>, context: EstablishTurnOrderContext) => {
            await actionQueue.enqueue({
                type: "EstablishTurnOrder",
                clientId: context.clientId,
            });
        }),
        sm.repeat(sm.transition((action: EstablishTurnOrderAction, context: EstablishTurnOrderContext) => {
            if (action.type !== "EstablishTurnOrder") {
                throw new Error("Invalid action type");
            }

            if (context.turnOrder.find((id) => id === action.clientId)) {
                throw new Error("Same client posted prime multiple times");
            }

            context.turnOrder.push(action.clientId); 
        }), players)
    ]);
}
```

Our `EstablishTurnOrderAction` is an alias for `BaseAction`, as it doesn’t
contain any additional information, just the client ID. The context contains our
`clientId` and the turn order array we need to populate.

The state machine posts our clientID as an action of type `EstablishTurnOrder`
action. Then for the given number of players, we expect an action of this type.
We check that incoming action is of this type, then we check we don’t see the
same action coming multiple times from the same client. Finally, we add the
received `clientId` to the `turnOrder` array.

And that’s it - once this executes, all clients will end up with the same
`turnOrder` array and will know whether it is their turn to act, or they should
be waiting for another client to take a turn.

We can extend this implementation to also establish a shared prime: each client
posts a prime, then the first one to arrive to others “wins” and becomes the
shared prime.

We’ll update our `EstablishTurnOrderAction` to include a prime:

```typescript
type SerializedPrime = string;

type EstablishTurnOrderAction = BaseAction & { prime: SerializedPrime };
```

We need to define a `SerializedPrime` (as a string) to work around the fact that
we can’t serialize `BigInt`s using `JSON.stringify()`, which is what we’re using
to serialize actions.

We extend our context to also include the shared prime:

```typescript
type EstablishTurnOrderContext = {
    clientId: ClientId;
    prime: bigint | undefined;
    turnOrder: ClientId[];
};
```

Our state machine also gets updated:

```typescript
function makeEstablishTurnOrderSequence(players: number) {
    return sm.sequence([
        sm.local(async (actionQueue: IQueue<EstablishTurnOrderAction>, context: EstablishTurnOrderContext) => {
            await actionQueue.enqueue({
                type: "EstablishTurnOrder",
                clientId: context.clientId,
                prime: BigIntUtils.bigIntToString(BigIntUtils.randPrime()),
            });
        }),
        sm.repeat(sm.transition((action: EstablishTurnOrderAction, context: EstablishTurnOrderContext) => {
            if (action.type !== "EstablishTurnOrder") {
                throw new Error("Invalid action type");
            }

            if (context.turnOrder.length === 0) {
                context.prime = BigIntUtils.stringToBigInt(action.prime);
            }

            if (context.turnOrder.find((id) => id === action.clientId)) {
                throw new Error("Same client posted prime multiple times");
            }

            context.turnOrder.push(action.clientId); 
        }), players)
    ]);
}
```

The only changes are:

1. When we enqueue our action, we generate a random prime and serialize it (we
   have a utility function that does this, which I won’t describe here).
2. If our `turnOrder` array is empty, meaning we just received the first action,
   we set the `prime` in the context.

With these changes, after we run this state machine we have both the turn order
and a prime all clients agree on.

To make calling this easier, we provide a function to initialize the context:

```typescript
function makeEstablishTurnOrderContext(
    clientId: ClientId
): EstablishTurnOrderContext {
    return {
        clientId,
        prime: undefined,
        turnOrder: [],
    };
}
```

Then putting it all together:

```typescript
export async function establishTurnOrder(
    players: number,
    clientId: ClientId,
    actionQueue: IQueue<BaseAction>
) {
    const context = makeEstablishTurnOrderContext(clientId);

    const establishTurnOrderSequence = makeEstablishTurnOrderSequence(players);

    await sm.run(establishTurnOrderSequence, actionQueue, context);

    return [context.prime!, context.turnOrder] as const;
}
```

We create a context, we create the state machine, then we run it. The function
returns the shared prime and the turn order.

## Summary

In this post we covered a few primitives or building blocks we can use for
building games:

* Creating a Fluid transport, and abstracting all the details under a
  `getLedger()` function. The code for this is in the `demo/transport` package,
  in
  [`container.ts`](https://github.com/vladris/mental-poker-toolkit/blob/main/demos/transport/src/container.ts).
* Upgrading the Fluid transport to a `SignedTransport` which signs outbound
  actions and verifies signatures of incoming actions. The code for this is in
  [`packages/primitives/upgradeTransport.ts`](https://github.com/vladris/mental-poker-toolkit/blob/main/packages/primitives/src/upgradeTransport.ts).
* Establish turn order for the players and agreeing on a shared large prime. The
  code for this is in
  [`packages/primitives/establishTurnOrder.ts`](https://github.com/vladris/mental-poker-toolkit/blob/main/packages/primitives/src/establishTurnOrder.ts).

With the primitives out of the way, in the next post we’ll look at the
high-level of modeling a game using the toolkit.
