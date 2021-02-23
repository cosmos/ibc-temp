# Application Modules

Upon successful authentication, the DID module will forward a `DelegatedServicePacket` to a third-party app module that will enact some app logic on behalf of the DID that authenticated.

Thus app modules connecting through the `did-app` port, must implement callbacks to handle receiving `DelegatedService` packets. But it should not send any packets back to the DID module, thus it does not need to handle acks or timeouts.

### Auth Module callbacks

```go
// OnChanOpenInit ensures that the app version is supported by 
// the app module,
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
    if !SupportedAppVersion(version) {
        return error
    }
    
    if order != UNORDERED {
        return error
    }
    return nil
}
```

```go
// OnChanOpenInit ensures that portID is `did-app`,
// ensures that the app version is supported by DID module,
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
    if !SupportedAppVersion(version) {
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
// OnRecvPacket supports receiving packets from the `did-app` port
// The app module may assume the packet is already authenticated by the DID
// and perform any custom app logic on that basis
func OnRecvPacket(
	ctx sdk.Context,
	packet channeltypes.Packet,
) (*sdk.Result, []byte, error) {
    switch packet.GetSourcePort() {
    case "did-app":
        servicePacket := decodeServicePacket()
        
        err := DoCustomAppLogic(servicePacket.DID, servicePacket.AppData)

        // App module may OPTIONALLY write acknowledgement.
        if err != nil {
            return result, unsuccessfulAck, nil
        } else {
            return result, successfulAck, nil

        }
    // the app module may also process packets from different ports
    default:
        return nil, nil, error
    }
}
```

```go
// OnAcknowledgementPacket is a no-op for the 'did-app' port of the app module.
// Since the app module does not send packets to DID module.
func OnAcknowledgementPacket(
    ctx sdk.Context,
    packet channeltypes.Packet,
    acknowledgement []byte,
) (*sdk.Result, error) {
    return nil, nil
}
```

```go
// OnTimeoutPacket is a no-op for the 'did-app' port of the app module.
// Since the app module does not send packets to DID module.
func OnTimeoutPacket(
    ctx sdk.Context,
    packet channeltypes.Packet,
) (*sdk.Result, error) {
    return nil
}
```