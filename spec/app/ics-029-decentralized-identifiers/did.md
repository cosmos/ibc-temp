# IBC DID Format

A common use case of decentralized identifiers is the creation of a standard digital identity that can authenticate with apps and services. This use case generally entails an **on-chain** registry of identity and associated verification methods and delegated capabilities that can then be used to authenticate and log-in to a variety of **off-chain** apps and services.

IBC DIDs will support this usecase, but will also enable authentication and capability delegation for **on-chain** apps.

Additionally, IBC DIDs will also support arbitrary authentication schemes by providing the ability to delegate verification to a third party module over IBC.

### Delegated Authentication

DIDs provide the ability to specify **verification methods** in the DID document that can be used to authenticate the DID subject.

The [DID specification](https://www.w3.org/TR/did-core/#verification-method-types) specifies a standard set of public key schemes that can be used to verify a DID subject. These verification methods will be natively supported by the DID module.

IBC DIDs will support arbitrary verification methods by supporting just a single new verification type: `DelegatedIBCAuthentication`.

The `DelegatedIBCAuthentication` type has two required fields:

`delegateTo`: This is the channel the DID module will delegate authentication to. The module on the other side of the channel will receive an authentication request from the DID module. If the module writes a successful acknowledgement, the DID module will interpret this as successful authentication.

`app-specific-verify-fields`: This field contains all the application specific parameters necessary to authenticate interaction with the DID. All parameters and data regarding authentication should be housed here, the module should simply be responsible for injesting these parameters along with user provided authentication and performing the logic necessary to authenticate. No state necessary for authentication should be stored in the counterparty module.

Example:

```yaml
"verification-methods": [
    {
        "id": "did:ibc:cosmos-hub:channel4:cns:ethermint#blockchain-governance",
        "type": "DelegatedIBCAuthentication",
        "controller": "did:ibc:cosmos-hub:channel4:cns:ethermint",
        "delegateTo": "channel3/gov",
        "app-specific-verify-fields": {
            "participation-threshold": 0.5,
            "yes-threshold": 0.5,
            "no-veto-threshold": 0.3,
        }
    },
],
```

### DID Controllers and Authentication Relationship

DIDs can associate different verification methods with different relationships. This allows DIDs to delegate authority over different services. 

IBC DIDs will regard the verification method associated with the `authentication` relationship as the ultimate verification method that is able to authenticate any packet on behalf of the DID subject. This allows the DID subject to maintain ultimate authority over every service.

DIDs also need a way to update themselves. The DID spec provides for a DID controller field which may or may not be the DID subject. The DID controller itself is a DID, and thus the DID module will authenticate any changes to the DID document using the verification method specified by the authentication relationship in the DID controller's document.

Example:

```yaml
"id": "did:ibc:cosmos-hub:cosmosaccaddr2",

# NOTE: DID controller is different than DID subject
# The verification method specified in controller's authentication relationship
# must pass in order to make any changes to this DID document.
"controller": "did:ibc:cosmos-hub:cosmosaccaddr1",

# The authentication relationship enables a verification method specified in this document
# to authenticate any outgoing packets on behalf of DID subject.
"authentication": "#verification-method-1",
```

### Delegated Services

With arbitrary verification methods and the authentication relationship, IBC DIDs already have the ability to define any authentication scheme they wish in order to send their application packets.

However, complex entities (such as organizations, blockchains, validator sets) may want to delegate the capability to send certain packets to another simpler entity (such as a trusted individual).

IBC DIDs enable this feature by providing DIDs the ability to associate a particular verification method with the right to send packets over a particular IBC channel. This allows DIDs to permission different entities to interact with different IBC apps on the DID subject's behalf.

IBC applications on the other hand, can use this fact to partition their IBC interactions over different ports in order to give DIDs the ability to assign different entities different capabilities within the same app by assigning them to different ports.

IBC DIDs introduce a new `Service` type: `DelegatedIBCService` that will enable delegating the ability to send packets over particular channels to different verification methods.

The `DelegatedIBCService` type has two required fields:

`serviceEndpoint`: This is the channel endpoint to which application packets will be sent. The module on the other side will receive an application packet and process it simply knowing that the DID subject had authenticated the packet; without knowing the specific delegated party that approved the packet or the particular authentication scheme.

`authentication`: This is the verification method that is given permission to send packets across this channel on the DID subject's behalf (in addition to the authentication relationship which can send packets anywhere). The verification method may be defined in this DID document or elsewhere.

```yaml
services: [
     {
        # id of the delegated service
        "id": "did:ibc:cosmos-hub:channel:module:module-specific-id#service1",
        "type": "DelegatedIBCService",
        # channel on which the delegated party may send packets
        "serviceEndpoint": "channel5/port3",
        # authentication that must pass in order to send the application packet along the `serviceEndpoint` channel
        "authentication": "did:ibc:cosmos-hub:channel:module:module-specific-id#delegatedAuthority1",
    },
],
```

### DID Document Format

DID: `did:ibc:cosmos-hub:channel:module:module-specific-id`

```yaml
{
"id": "did:ibc:cosmos-hub:channel:module:module-specific-id",

"controller": "did:ibc:cosmos-hub:channel:module:module-specific-id",

"verification-methods": [
    # Some verification methods are delegated to other authentication modules.
    # DID will send authentication data along with any module-specific params specified in a `DelegateAuth` Packet. Receiving a successful ACK on the packet is interpreted as successful authentication.
    {
        "id": "did:ibc:cosmos-hub:channel:module:module-specific-id#baseAuthority"
        "type": "DelegatedIBCAuthentication",
        "controller": "did:ibc:cosmos-hub:channel:module:module-specific-id",
        "delegateTo": "channel3/port1"
        "app-specific-verify-fields": {
            ...
        }
    },
    {
        "id": "did:ibc:cosmos-hub:channel:module:module-specific-id#delegatedAuthority1",
        "type": "DelegatedIBCAuthentication",
        "controller": "did:cosmos-hub:channel:module:module-specific-id",
        "delegateTo": "channel4/port2"
        "app-specific-verify-fields": {
            ...
        }
    },
    {
        "id": "did:ibc:cosmos-hub:channel:module:module-specific-id#delegatedAuthority2",
        "type": "DelegatedIBCAuthentication",
        "controller": "did:cosmos-hub:channel:module:module-specific-id",
        "delegateTo": "channel1/port2",
        "app-specific-verify-fields": {
            ...
        }
    },
    {
      "id": "did:ibc:cosmos-hub:channel:module:module-specific-id#native-key1",
      "type": "Ed25519VerificationKey2018",
      "controller": "did:example:123",
      "publicKeyBase58": "H3C2AVvLMv6gmMNam3uVAjZpfkcJCwDwnZn6z3wXmqPV"
    }
    ...
],

# The verification method specified in authentication relationship is able to authenticate any packet
"authentication": "did:ibc:cosmos-hub:channel:module:module-specific-id#baseAuthority",

"service": [
    {
        "id": "did:ibc:cosmos-hub:channel:module:module-specific-id#service1",
        "type": "DelegatedIBCService",
        "serviceEndpoint": "channel5/port3",
        "authentication": "did:ibc:cosmos-hub:channel:module:module-specific-id#delegatedAuthority1",
    },
    {
        "id": "did:ibc:cosmos-hub:channel:module:module-specific-id#service2",
        "type": "DelegatedIBCService",
        "serviceEndpoint": "channel7/port2",
        "authentication": "did:ibc:cosmos-hub:channel:module:module-specific-id#delegatedAuthority2",
    },
    {
        "id": "did:ibc:cosmos-hub:channel:module:module-specific-id#service3",
        "type": "DelegatedIBCService",
        "serviceEndpoint": "channel9/port12",
        "authentication": "did:ibc:cosmos-hub:channel:module:module-specific-id#native-key1",
    },
]
}
```

### Full DID Document Example

### Examples

Here is a DID that defines how a chain interacts with the Chain-Naming-Service module over IBC. Here blockchain governance maintains the DID document and can send arbitrary packets to the CNS and other modules. However, the document also specifies delegated authorities that can only send packets across specific channels. DID subjects can use this to specify which authorities can send packets across which channels. Applications can use this fact to listen on multiple ports and thus provide the ability for different authorities to authorize different interactions for the same subject.

DID: `did:ibc:cosmos-hub:channel3:cns:ethermint`

```yaml
{
"id": "did:ibc:cosmos-hub:channel:channel4:cns:ethermint",

# The verification method in the authentication relationship
# of the controller can make updates to this DID document.
# NOTE: Controller can be different than Subject
"controller": "did:ibc:cosmos-hub:channel4:cns:ethermint",

"verification-methods": [
    # This is the base verification method that controls this DID
    # and can authenticate any packet for this DID.
    {
        "id": "did:ibc:cosmos-hub:channel4:cns:ethermint#blockchain-governance",
        "type": "DelegatedIBCAuthentication",
        "controller": "did:ibc:cosmos-hub:channel4:cns:ethermint",
        "delegateTo": "channel3/gov",
        # NOTE: These fields do not necessarily need to exist in this document,
        # they can be hardcoded into the governance module.
        ## However, doing this might allow the DID document to specify
        # different required thresholds for different services.
        "app-specific-verify-fields": {
            "participation-threshold": 0.5,
            "yes-threshold": 0.5,
            "no-veto-threshold": 0.3,
        }
    },
    {
        "id": "did:ibc:cosmos-hub:channel4:cns:ethermint#blockchain-gov2",
        "type": "DelegatedIBCAuthentication",
        "controller": "did:cosmos-hub:channel3:gov",
        "delegateTo": "channel3/gov"
        # Here is an example of using the same authentication module
        # with different parameters to create a more "lenient" governance
        # mechanism.
        "app-specific-verify-fields": {
            "participation-threshold": 0.3,
            "yes-threshold": 0.2,
            "no-veto-threshold": 0.1,
        },
    },
    # Channel1/port2 implements a fancy multisig protocol
    {
        "id": "did:ibc:cosmos-hub:channel3:cns:ethermint#delegatedAuthority2",
        "type": "DelegatedIBCAuthentication",
        "controller": "did:cosmos-hub:channel4:cns:ethermint",
        "delegateTo": "channel1/port2",
        "app-specific-verify-fields": {
            "pk1": x55...,
            "power1": 30,
            "pk2": x32...,
            "power2":, 45,
            "pk3": x22...,
            "power3": 80,
            "threshold": 70,
        }
    },
    # Channel8/port2 is a channel to the chain that owns the CNS domain.
    # It will send a packet if the current validator set commits to the packet data and destination at a specific key.
    # It needs no app-specific-verify-fields and will use PacketFlow2 depicted below.
    {
        "id": "did:ibc:cosmos-hub:channel3:cns:ethermint#validatorAuthority",
        "type": "DelegatedIBCAuthentication",
        "controller": "did:cosmos-hub:channel4:cns:ethermint",
        "delegateTo": "channel8/port2",
    },
    # Channel4/port7 is a channel to a solo-machine
    # There are no app-specific verify fields. The solo-machine is trusted to authorize the party on a private server and send an IBC packet across this channel.
    # The server might implement a traditional username/password scheme to authenticate the subject before sending the packet
    # This verification method would be used with PacketFlow 2 depicted below.
    {
        "id": "did:ibc:cosmos-hub:channel3:cns:ethermint#loginAuthority",
        "type": "DelegatedIBCAuthentication",
        "controller": "did:cosmos-hub:channel4:cns:ethermint",
        "delegateTo": "channel4/port7",
    },
    # This is a native authentication scheme. A simple public key which the DID module can verify on its own.
    {
      "id": "did:ibc:cosmos-hub:channel:module:module-specific-id#native-key1",
      "type": "Ed25519VerificationKey2018",
      "controller": "did:example:123",
      "publicKeyBase58": "H3C2AVvLMv6gmMNam3uVAjZpfkcJCwDwnZn6z3wXmqPV"
    },
    ...
],

# The verification method specified in authentication relationship is able to authenticate any packet
"authentication": "did:ibc:cosmos-hub:channel:module:module-specific-id#baseAuthority",

"services": [
    # A lenient blockchain governance scheme is used to change light client fields in CNS mapping.
    {
        "id": "did:ibc:cosmos-hub:channel3:cns:ethermint#ChangeClientFields",
        "type": "DelegatedIBCService",
        "serviceEndpoint": "channel5/port3",
        "authentication": "did:ibc:cosmos-hub:channel:module:module-specific-id#blockchain-gov-2",
    },
    {
        # The current validator set is able to change the RPC fields
        "id": "did:ibc:cosmos-hub:channel:module:module-specific-id#ChangeRPC",
        "type": "DelegatedIBCService",
        "serviceEndpoint": "channel9/port12",
        "authentication": "did:ibc:cosmos-hub:channel:module:module-specific-id#validatorAuthority",
    },
    {
        # A multisig over the core developers can change the github fields in CNS mapping
        "id": "did:ibc:cosmos-hub:channel:module:module-specific-id#ChangeGitHub",
        "type": "DelegatedIBCService",
        "serviceEndpoint": "channel7/port2",
        "authentication": "did:ibc:cosmos-hub:channel:module:module-specific-id#delegatedAuthority2",
    },
    {
        # The founder of the project can change his about page
        "id": "did:ibc:cosmos-hub:channel:module:module-specific-id#AboutFounder",
        "type": "DelegatedIBCService",
        "serviceEndpoint": "channel9/port12",
        "authentication": "did:ibc:cosmos-hub:channel:module:module-specific-id#native-key1",
    },
    ...
]
}
```

NOTE: The names of the ids may not be human meaningful. This is a TBD part of the design. They're left here meaningful while still respecting the DID URI spec for readability.

### Herd Privacy

An important notion in DIDs is the notion of herd privacy, ie the idea that indistinguishable (DID, DID Document) pairs provides some level of privacy for each indistinguishable (DID, DID Document) pair.

However, this would prevent the use of arbitrary authentication schemes and the ability to authorize IBC interactions using the DID document since the authentication schemes used and the types of IBC services supported by a DID subject leaks some information as to what the subject might be. The above example is pretty obviously describing a blockchain.

Thus, any DID that uses `DelegatedIBCAuthentication` or `DelegatedIBCService` should not expect herd privacy. DID subjects intending to preserve privacy should stick to native verification methods and not offer any services (apart from perhaps a third-party mediator service desribed [here](https://github.com/interNFT/nft-rfc/pull/11)). Furthermore, the DID subject should take care that their private DID(s) is not publically correlated with a IBC-enabled DID that they might use.
