# Packets and Messages

### RequestedServiceMessage

Users may send packets to application modules on behalf of a DID by sending a `RequestServiceMessage` to the DID module.

The message that users send to the DID module must contain the DID, the requested verification method, the requested service, and authenticated application data.

```go
type RequestServiceMessage {
    // DID that will be resolved to DID document
    DID string
    // Verification ID that will reference the way to authenticate user data
    VerificationID string
    // Service ID identifies the IBC app that will receive the user data after it has been authenticated
    ServiceID string
    // Data to be sent by app, this is only meant to be understood by the app module it will eventually be sent to.
    AppData []byte
    // User provided authentication to be verified given the verification method specified by VerificationID
    // This is only meant to be understood by the module that authenticates the request 
    // (either DID module in native authentication or the delegated auth module)
    Authentication []byte
}
```

### Authenticating a RequestedServiceMessage

Upon receiving a `RequestServiceMessage`, the DID module must perform the following steps in order to authenticate the request.

1. Resolve DID document from provided DID
2. Check if verification method in DID's `authentication` relationship is same as provided verification ID.
3. Resolve serviceID from DID document and check if verification method in service's `authentication` field matches provided verification ID.
4. If neither 2. or 3. match then return an error
5. If verification method referenced in verification ID has a native type, then DID module authenticates the `AppData` using the `Authentication` data with the appropriate authentication schema.
6. If verification method referenced in verification ID has type `DelegatedIBCAuthentication`, then DID module sends a `DelegatedAuthPacket` and thus delegates the authentication to the appropriate third-party auth module specified by the `delegateTo` field.
7. If either 5. or 6. return an error, then return an error to user. Upon success, forward the app data to the requested app module on behalf of the DID.

### Delegated Authentication

The DID module delegates authentication to third-party auth modules using IBC. A user may send a message to the DID module asking to authenticate a request to send an application packet on the DID's behalf.

If the verification type is a natively supported scheme, then the DID module will verify the authentication on its own. In the case, where the verification type is `DelegatedIBCAuthentication`, then the DID module forwards the authentication data to the `delegateTo` module for verification and waits for the acknowledgment before proceeding.

```go
type DelegateAuthPacketData {
    DID string
    Sequence uint64 // sequence for this DID subject
    // Marshalled AppSpecificVerifyFields provided in DID document
    AppSpecificVerifyFields []byte
    // App Data intended to be sent to app module
    AppData []byte
    // Authentication data provided by user must be understandable
    // by third-party auth module.
    Authentication []byte
}
```

The authentication module must understand the app specific verify fields along with the authentication data in order to verify that the app data has been properly authorized.

Upon processing the authentication data, the authentication module will write an acknowledgement that encodes whether the app data was successfully authenticated.

```go
type DelegateAuthPacketAck {
    // The subject and sequence will specify which service request this ack is for
    DID string
    Sequence uint64
    Success bool
}
```

The DID and sequence together uniquely refer to a service request received by the DID module. Thus when the Acknowledgment comes back from the auth module, the DID module can correctly route the app data to its intended destination.

### Delegated Service

Upon successful authentication, the DID module can then send the app data to the requested module on the DID's behalf. To do this it perfoms the following steps:

1. Once authenticated, resolve DID document from DID in original message.
2. Read ServiceID from DID document and retrieve `serviceEndpoint` field.
3. Send `DelegatedServicePacket` to channel specified by service endpoint.

```go
type DelegatedServicePacketData {
    DID string
    // DID module has authenticated that DID truly
    // sent app data.
    // Receiving module does not need to know how.
    AppData []byte
}
```

The service on the other end of the channel, is responsible for decoding the application data and processing it. The service MAY choose to send an ACK back to the DID module, though it is not required. The DID module will simply log the success or failure of the packet in the events.

### Packet Flows

This design enables two possible packet flows depending on context. All packet flows must be triggered by an end user. That end user may choose to interact with the DID module, which will authenticate and then forward an IBC packet to the application module. Or they can send a msg directly to the authenticating module, which will send a packet through the DID module and then to the application module.

Use case 1:
```
User Msg -> DID Module ----optional DelegatedAuthPacket ---> Auth module
                |                                                 |
                |                                                 |
User<--Result---|<----------ACK {Success/Failure}-----------------|
                |
            if auth succeeds
                |
                |--------------Send AppPacket---------------> App module
                |                                                  |
                |                                                  |
User<--Result---|<----------ACK {Success/Failure}------------------|
```

Use case 2:

```
Module1 ---AppPacket--> DID Module ----AppPacket---------------> Module2
                                        |                           |
                                        |                           |
<---Result----|<-----ACK{Success/Fail}--|                           |
                                                                    |
<---Result----|<-----ACK{Success/Fail}--|<----ACK{Success/Fail}-----|
```
