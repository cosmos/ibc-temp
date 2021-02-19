# DID Authentication Modules

The DID module may delegate authentication to third-party authentication modules through the `did-auth` port. The DID module may send `DelegatedAuthentication` packets to the auth module, if the users are requesting services through the DID module. If the users are interacting directly with the auth module, the auth module may directly send a `DelegatedService` packet to the DID module.

### Send Packet to DID Module

Depending on specific authentication module logic, the auth module may send the packet in a handler, and endblocker, or begin-blocker. 

Once the user interaction(s) have been properly authenticated, the `DelegatedSendPacket` can be sent like so:

```go
// After doing custom authentication of user interactions
packet := DelegatedServicePacket{
    DID: userRequested.DID,
    ServiceID: userRequested.ServiceID,
    AppData: userRequested.AppData,
}
sendPacketToDidModule(packet)
```

### Auth Module callbacks

```go
// OnChanOpenInit ensures that the auth version is supported by 
// the auth module,
// and that the channel is UNORDERED.
func OnChanOpenInit(
    ctx sdk.Context,
    order channeltypes.Order,
    connectionHops []string,
    portID string,
    channelID string,
    channelCap *capabilitytypes.Capability,
    counterparty channeltypes.Counterparty,
    version string,
) error {
    if !SupportedAuthVersion(version) {
        return error
    }
    
    if order != UNORDERED {
        return error
    }
    return nil
}
```

```go
// OnChanOpenInit ensures that portID is `did-auth`,
// ensures that the auth or app version is supported by DID module,
// and that the channel is UNORDERED.
func OnChanOpenTry(
    ctx sdk.Context,
    order channeltypes.Order,
    connectionHops []string,
    portID,
    channelID string,
    channelCap *capabilitytypes.Capability,
    counterparty channeltypes.Counterparty,
    version,
    counterpartyVersion string,
) error {
    if !SupportedAuthVersion(version) {
        return error
    }
    
    if counterpartyVersion != version {
        return error
    }
    if order != UNORDERED {
        return error
    }
    return nil
}
```

```go
// OnChanOpenAck ensures that counterparty version is the same as version.
func OnChanOpenAck(
    ctx sdk.Context,
    portID,
    channelID string,
    counterpartyVersion string,
) error {
    if counterpartyVersion != version {
        return error
    }
    return nil
}
```

```go
// OnChanOpenConfirm is a no-op
func OnChanOpenConfirm(
    ctx sdk.Context,
    portID,
    channelID string,
) error {
    return nil
}
```

```go
// OnChanCloseInit always returns an error to prevent user-initiated channel closure.
func OnChanCloseInit(
    ctx sdk.Context,
    portID,
    channelID string,
) error {
    return error
}
```

```go
// OnChanCloseConfirm is a no-op to allow channel end to close
// if counterparty closes their channel end.
func OnChanCloseConfirm(
    ctx sdk.Context,
    portID,
    channelID string,
) error {
    return nil
}
```

```go
// OnRecvPacket only supports receiving packets from the `did-auth` port
// since auth modules may directly send packets to DID module in the
// packet flow 2 case.
// In this case, the DID module checks that the sending module is authorized to send the packet
// and then forwards it along.
func OnRecvPacket(
	ctx sdk.Context,
	packet channeltypes.Packet,
) (*sdk.Result, []byte, error) {
    switch packet.GetSourcePort() {
    case "did-auth":
        authPacket := decodeAuthPacket()
        
        err := DoCustomVerification(authPacket.AppSpecificVerifyFields, authPacket.AppData, authPacket.AuthData)

        if err != nil {
            return result, successfulAck, nil
        } else {
            return result, unsuccessfulAck, nil
        }
    default:
        return nil, nil, error
    }
}
```

```go
// OnAcknowledgementPacket will listen for acks on the did-auth port and upon receiving a successful ack from the auth module,
// the did module will forward the app packet to the requested app module.
// It will delete the stored service request on a failed ack.
// Any acks arriving from the app module will simply be logged for the end user.
func OnAcknowledgementPacket(
    ctx sdk.Context,
    packet channeltypes.Packet,
    acknowledgement []byte,
) (*sdk.Result, error) {
    // Do custom acknowledgement logic
    return nil, nil
}
```

```go
// OnTimeoutPacket will delete a service request if the DelegatedAuthPacket times out
func OnTimeoutPacket(
    ctx sdk.Context,
    packet channeltypes.Packet,
) (*sdk.Result, error) {
    // Do custom timeout logic
    return nil
}
```