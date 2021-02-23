# IBC Enabled DIDs

### Background Reading

https://www.w3.org/TR/did-core/

### Context

As the IBC ecosystem evolves, there will be many different applications and many different entities that will want to interact with those IBC applications. The basic use case is that a single user wants to authenticate an IBC packet being sent. This is simple enough to implement with the `x/auth` module in the SDK. Even a well-defined, constant set of users is relatively simple to implement using the MultiSig provided in the `x/auth` module. These authentication schemes may be sufficient for apps like the [ICS-20](https://github.com/cosmos/ics/tree/master/spec/ics-020-fungible-token-transfer) token transfer module.

However, the set of entities who wish to send IBC packets will naturally become more complex and diverse as the set of IBC applications becomes more complex and diverse. For example, a tendermint validator set may want to be able to authenticate and send IBC packets. Blockchain governance may also want to send IBC packets to other chains on the basis of passing a proposal. All of these entities will need to implement their own form of authentication that must pass in order to send the IBC packet. If each authentication scheme is implemented in a different way, then IBC applications that need to process an "authenticating party" before sending the IBC packet will have to explicitly enable each authentication scheme.

To take the transfer case; in order to allow blockchain governance or a validator set to authorize a transfer packet, ICS-20 either has to implement these authentication schemes itself. Or each authentication module implemented as a standalone module must include a transfer keeper which will send the packet on successful authentication. This does not scale as the number of authentication schemes and applications wish to interoperate.

Instead a uniform authentication layer that sits between specific authentication schemes and their end applications will provide a uniform interface across which entities can authenticate interactions with IBC applications. This would allow arbitrary authentication schemes to implement the interface for authenticating via the uniform authentication layer without understanding the IBC applications the user data is intended for, and applications can simply receive successfully authenticated user data from the universal authentication layer without understanding how the data was authenticated.

At the same time, as the IBC ecosystem develops; there is also a developing ecosystem around DIDs. A DID is a decentralized identifier identifying a subject (e.g. internet user, land title, image, etc) that resolves to a DID document, which can contain information on how to authenticate interactions from the subject and specifies available services related to the DID subject (Eg twitter account, inbox, etc).

The DID document for an internet user might include their public keys and which public key should be used for which interaction. Perhaps the DID document specifies an ED25519 PublicKey for authenticating web log-ins, and a JsonWebKey2020 for negotiating secret keys for communication. The (DID, DID Document) pair is maintained on a public ledger and allows the DID document to be updated, allowing other parties to authenticate interactions from the DID subject even as public keys are rotated.

This again works fine in the case of a single user or a well-defined constant set of users, since these verification methods can be encoded in a public key or multisig. However, these methods make it difficult for more complex entities like a rotating validator set or blockchain governance to identify itself with a DID and define specific verification methods for specific purposes.

We can use IBC to provide more flexibility in how DIDs authenticate and verify subject interactions AND use that DID module to serve as a uniform authentication layer for IBC applications.

### Design Goals:

1. The DID module should be able to act as a uniform authentication layer for arbitrary entities to authenticate and interact with on-chain and off-chain services.
2. Allow modules to implement arbitrary authentication logic on behalf of DID subject, with the DID subject authorizing different verification methods for different purposes.
3. Allow the DID subject to interact and authenticate with arbitrary applications, and send arbitrary application data along to the application.
