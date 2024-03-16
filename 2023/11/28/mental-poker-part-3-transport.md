# Mental Poker Part 3: Transport

Now that my [LLM book is done](https://vladris.com/blog/2023/11/03/large-language-models-at-work-rtm.html),
I can get back to the Mental Poker series. A high-level overview can be found
[here](https://vladris.com/blog/2023/02/18/mental-poker-part-0-an-overview.html).
In the previous posts we covered
[cryptography](https://vladris.com/blog/2023/03/14/mental-poker-part-1-cryptography.html)
and [a Fluid append-only list data
structure](https://vladris.com/blog/2023/06/04/mental-poker-part-2-fluid-ledger.html).
We’ll be using the append-only list (we called this `fluid-ledger`) to model
games.

An append-only list should be all that is needed to model turn-based games: each
*turn* is an element added to the list. In this post, we’ll stitch things
together and look at the transport layer for our games.

## Transport

Our basic transport interface is very simple:

```typescript
declare interface ITransport<T> {
    getActions(): IterableIterator<T>;

    postAction(value: T): Promise<void>;

    once(event: "actionPosted", listener: (value: T) => void): this;
    on(event: "actionPosted", listener: (value: T) => void): this;
    off(event: "actionPosted", listener: (value: T) => void): this;
}
```

For some type `T`, we have:

* A `getActions()`, which returns an iterator over all values (of type `T`)
  posted so far.
* A `postAction()`, which takes a value of type `T` and an `actionPosted` event
  which fires whenever any of the clients posts an action (this relies on the
  Fluid data synchronization).
* And the standard `EventEmitter` methods.

We'll cover why we call these values *actions* in a future post.

The basic implementation of this on top of the `fluid-ledger` distributed data
structure looks like this:

```typescript
class FluidTransport<T> extends EventEmitter implements ITransport<T> {
    constructor(private readonly ledger: ILedger<string>) {
        super();
        ledger.on("append", (value) => {
            this.emit("actionPosted", JSON.parse(value) as T);
        });
    }

    *getActions() {
        for (const value of this.ledger.get()) {
            yield JSON.parse(value) as T;
        }
    }

    postAction(value: T) {
        return Promise.resolve(this.ledger.append(JSON.stringify(value)));
    }
}
```

The constructor takes an `ILedger<string>` (this is the interface we looked at
in [the previous post](https://vladris.com/blog/2023/06/04/mental-poker-part-2-fluid-ledger.html)).

It hooks up an event listener to the ledger's `append` event to in turn trigger
an `actionPosted` event. We also convert the incoming value from `string` to `T`
using `JSON.parse()`.

Similarly, `getActions()` is a simple wrapper over the underlying ledger, doing
the same conversion to `T`.

Finally, the `postAction()` does the reverse - it converts from `T` to a string
and appends the value to the ledger.

With this in place, we abstracted away the Fluid-based transport details. We
will separately set up a Fluid container and establish connection to other
clients (in a future post), then take the `ILedger` instance, pass it to
`FluidTransport`, and we are good to go.

We can model games on top of just these two primitives: `postAction()` and
`actionPosted`. Whenever we take a turn, we call `postAction()`. Whenever any
player takes a turn, the `actionPosted` event is fired.

Since we’re designing Mental Poker, which takes place in a zero-trust
environment, let’s make sure our transport is secure.

## Signature verification

Signature verification allows us to ensure that in a multiplayer game, players
can’t spoof each other, meaning Alice can’t pretend she is Bob and post an
action on Bob’s behalf for other clients to misinterpret.

Note in a 2-player game this is not strictly needed if we trust the channel: we
know that if a payload was not sent by us, it was sent by the other player. But
in games with more players, we need to protect against spoofing. Signatures are
also useful in case we don’t trust the channel - maybe it’s supposed to be a
2-player game but a third client gets access to the channel and starts sending
messages.
 
We will implement this using public key cryptography. The way this works is each
player generates (locally) a public/private key pair. They broadcast the public
key to all other players. Then they can sign any message they send with their
private key and other players can validate the signature using the public key.
Nobody else can sign on their behalf, since the private key is kept private.

I won’t go into deeper detail here, since this is very standard public key
cryptography. In fact, I didn’t even cover this in the blog post covering
[cryptography](https://vladris.com/blog/2023/03/14/mental-poker-part-1-cryptography.html)
for Mental Poker for this reason. There, I focused on the commutative SRA
encryption algorithm. Unlike SRA, which we had to implement by hand, signature
verification is part of the standard [Web Crypto
API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Crypto_API). Let’s
implement signature verification on top of this.

First, we need to model a public/private key pair:

```typescript
// Keys are represented as strings
export type Key = string;

// Public/private key pair
export type PublicPrivateKeyPair = {
    publicKey: Key;
    privateKey: Key;
};
```

A key is a string. We model the key pair as `PublicPrivateKeyPair`, a type
containing two keys. Here’s how we generate the key pair using the Web Crypto
API:

```typescript
import { encode, decode } from "base64-arraybuffer";

async function generatePublicPrivateKeyPair(): Promise<PublicPrivateKeyPair> {
    const subtle = crypto.subtle;
    const keys = await subtle.generateKey(
        {
            name: "rsa-oaep",
            modulusLength: 4096,
            publicExponent: new Uint8Array([1, 0, 1]),
            hash: "sha-256",
        },
        true,
        ["encrypt", "decrypt"]
    );

    return {
        publicKey: encode(await subtle.exportKey("spki", keys.publicKey)),
        privateKey: encode(
            await subtle.exportKey("pkcs8", keys.privateKey)
        ),
    };
}
```

We use `subtle` to generate our key pair and return both public and private keys
as base64-encoded strings.

We can similarly rely on `subtle` for signing. The following function takes a
string payload and signs it with the given private key. The response is the
base64-encoded signature.

```typescript
async function sign(
    payload: string,
    privateKey: Key
): Promise<string> {
    const subtle = crypto.subtle;

    const pk = await subtle.importKey(
        "pkcs8",
        decode(privateKey),
        { name: "RSA-PSS", hash: "SHA-256" },
        true,
        ["sign"]
    );

    return encode(
        await subtle.sign(
            { name: "RSA-PSS", saltLength: 256 },
            pk,
            decode(payload)
        )
    );
}
```

First, we import the given `privateKey`, then we call `subtle.sign()` to sign
the base64-decoded `payload`. We re-encode the signature to base64 and return it
as a string.

Finally, this is how we verify signatures:

```typescript
async function verifySignature(
    payload: string,
    signature: string,
    publicKey: Key
): Promise<boolean> {
    const subtle = crypto.subtle;

    const pk = await subtle.importKey(
        "spki",
        decode(publicKey),
        { name: "RSA-PSS", hash: "SHA-256" },
        true,
        ["verify"]
    );

    return subtle.verify(
        { name: "RSA-PSS", saltLength: 256 },
        pk,
        decode(signature),
        decode(payload)
    );
}
```

Here, we import the given `publicKey`, then we use `subtle.verify()`. For
signature verification, we pass in a `signature` and the `payload` that was
signed (decoded from base64). This API returns `true` if the signature matches,
meaning it was indeed signed with the private key corresponding to the public
key we provided.

Again, I won’t go deep into the `subtle` APIs as they are standard and very well
documented. The main takeaway is now we have 3 APIs:

* `generatePublicPrivateKeyPair()` to generate key pairs.
* `sign()` to sign a payload.
* `verify()` to validate the signature.

We’ll put these in the `Signing` namespace.

Now let’s layer this cryptography over our `FluidTransport`.

## Signed transport

Now that we have our Fluid-based implementation of the `ITransport` interface
and signature verification functions, we’ll provide another implementation of
this interface that handles signature verification.

First, we need a generic `Signed` type:

```typescript
type clientId = string;

type Signed<T> = T & { clientId?: ClientId; signature?: string };
```

This takes any type `T` and extends it with an optional `clientId` and
`signature`. We’ll represent client IDs as strings.

Now we can decorate any payload in our transport with these optional `clientID`
and `signature`, which we can then validate using the functions we just
implemented. The reason these are optional is that we have states when signing
is unavailable: before clients exchange public keys. During the key exchange
steps, no message can be signed, since no client knows the public key of any
other client. These messages can’t be signed. Once keys are exchanged, all
subsequent messages *should* be signed, and we’ll enforce that in
`SignedTransport`.

We also need a `KeyStore`. This keeps track of which public key belongs to each
client, to help with our signature verification (meaning we keep track of which
public key is Alice’s, which one is Bob’s and when we get a message from Alice
we know which key to use to verify authenticity).

```typescript
type KeyStore = Map<ClientId, Key>;
```

We also need a `ClientKey` type, representing a single client ID/private key
pair:

```typescript
export type ClientKey = { clientId: ClientId; privateKey: Key };
```

With these additional type definitions in place, we can start building our
`SignedTransport<T>`. This is a decorator that takes an `ITransport<Signed<T>>`.
We’ll first look at the constructor:

```typescript
class SignedTransport<T> extends EventEmitter implements ITransport<T> {
    constructor(
        private readonly transport: ITransport<Signed<T>>,
        private readonly clientKey: ClientKey,
        private readonly keyStore: KeyStore
    ) {
        super();
        transport.on("actionPosted", async (value) => {
            this.emit("actionPosted", await this.verifySignature(value));
        });
    }

/* ... */
```

This new class has 3 private properties. Let’s discuss them in turn.

`transport` is our underlying `ITransport<Signed<T>>`. The idea is we can
instantiate a `FluidTransport` (or other transport if needed, though for this
project I have no plans of using another transport than Fluid), then pass it in
the constructor here. Then `SignedTransport` will use the provided instance for
`postAction()` and `actionPosted`, simply adding signature verification over it.

The `clientKey` should be this client’s ID and private key. This class is not
concerned with key generation, just signature and verification, so we’ll have to
generate the key pair somewhere else and pass it. We’ll use this to sign our
outgoing payloads.

We also pass in a `keyStore`. This should have the client ID to public key
mapping for all players in the game. We use this to figure out which public key
to use to validate each posted action.

### Existing actions

`getActions()` simply calls the underlying transport - we are not doing
signature verification on existing messages, since they were likely sent before
the signed transport was created and cannot be verified.

```typescript
*getActions() {
    for (const value of this.transport.getActions()) {
        yield value;
    }
}
```

We only validate incoming actions.

### Incoming actions

The constructor body hooks up the `actionPosted` event to the `transport`’s
`actionPosted`. So whenever the underlying transport fires the event, the
`SignedTransport` will also fire an `actionPosted` event. But instead of just
passing `value` through, we call `verifySignature()` on the `value` first.

Let’s look at `verifySignature` next (this is also part of the `SignedTransport`
class):

```typescript
private async verifySignature(value: Signed<T>): Promise<T> {
    if (!value.clientId || !value.signature) {
        throw Error("Message missing signature");
    }

    // Remove signature and client ID from object and store them
    const clientId = value.clientId;
    const signature = value.signature;

    delete value.clientId;
    delete value.signature;

    // Figure out which public key we need to use
    const publicKey = this.keyStore.get(clientId);

    if (!publicKey) {
        throw Error(`No public key available for client ${clientId}`);
    }

    if (
        !(await Signing.verifySignature(
            JSON.stringify(value),
            signature,
            publicKey
        ))
    ) {
        throw new Error("Signature validation failed");
    }

    return value;
}

/* ... */
```

Since `value` is a `Signed<T>`, we should have a `clientId` and a `signature`.
We throw an exception if we can’t find them.

Next, we clean up `value` and remove the `clientId` and `signature` from the
object. As we return this to other layers in our stack, they no longer need this
as we’re handling signature verification here.

We then try to retrieve the public key of the client from the `keyStore`. We
again throw in case we don’t have the key.

We use the `verifySigntature()` function we implemented earlier to ensure the
signature is valid. We throw if not.

At this point, we guaranteed that the payload is coming from the client claiming
to have sent it. If Alice tries to forge a message and pretend it’s coming from
Bob, she wouldn’t be able to produce a valid Bob signature (since only Bob has
access to his private key). Such a message would not make it past this function.

If no exceptions were thrown, this function returns a `value` (with signature
cleaned up), ready to be processed by other layers.

### Outgoing actions

Let’s now look at adding signatures to `postAction()`. `signAction()` is another
private class member handling signing:

```typescript
private async signAction(value: T): Promise<Signed<T>> {
    const signature = await Signing.sign(
        JSON.stringify(value),
        this.clientKey.privateKey
    );

    return {
        ...value,
        clientId: this.clientKey.clientId,
        signature: signature,
    };
}

/* ... */
```

We call the `sign()` function we implemented earlier in this post, passing it
the stringified `value` and our client’s private key. We then extend `value`
with the corresponding `clientId` and `signature`.

The `postAction()` implementation uses this function for signing, before calling
the underlying’s transport `postAction()`.

```typescript
async postAction(value: T) {
    this.transport.postAction(await this.signAction(value));
}
```

We now have the full implementation of `SingedTransport`.

## Summary

We started with a simple `FluidTransport` that uses a  `fluid-ledger` to
implement the `postAction()` function and `actionPosted` event, which we need
for modeling turn-based games.

Next, we looked at signing and signature verification using `subtle`.

Finally, we implemented `SingedTransport`, a decorator over another transport
that adds signature singing and verification.

The idea is we start with a `FluidTransport` and perform a key exchange, where
each client generates a public/private key pair and broadcasts their ID and
public key. Clients store all these in a `KeyStore`. Once the key exchange is
done, we can initialize a `SignedTransport` that wraps the original
`FluidTransport` and transparently handles signatures.

At this point we have all the pieces in place to start looking at semantics: we
can exchange data between clients, we can authenticate exchanged messages, and
we have the cryptography primitives for Mental Poker (commutative encryption).
In the next post we’ll look at a state machine that we can use to implement game
semantics.

The code covered in this post is available on GitHub in the [`mental-poker-toolkit` repo](https://github.com/vladris/mental-poker-toolkit/).
`FluidTransport` is implemented under [`packages/fluid-transport`](https://github.com/vladris/mental-poker-toolkit/tree/main/packages/fluid-client),
`SignedTransport` is under [`packages/signed-transport`](https://github.com/vladris/mental-poker-toolkit/tree/main/packages/signed-transport),
and the signing functions can be found in [`packages/cryptography/src/signing.ts`](https://github.com/vladris/mental-poker-toolkit/blob/main/packages/cryptography/src/signing.ts).

**Note:** Since writing this post, the code was refactored so `SignedTransport`
doesn't take a direct dependency on the cryptography package, rather signing
and signature verification is now passed as a `ISignatureProvider` interface.
