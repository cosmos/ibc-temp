# Channel Establishment and Callbacks

### Channel Establishment

Before `DelegatedAuth` packets or `DelegatedService` packets can be sent across their respective `delegateTo` and `serviceEndpoint` channels, the channels must be established between the DID module and the respective counterparty module.

The DID module must differentiate between channels connecting to counterparty auth modules and channels connecting to counterparty app modules since they send different packets and have different callback logic.

Thus the DID module reserves two ports: `did-auth` and `did-app` to service these two separate communication modes. Channels that connect to auth modules must use the `did-auth` port, and channels that connect to app modules must use the `did-app` port.

Any user or relayer may establish a connection between a counterparty module and one of these two ports. Once established, any channel connected to the `did-auth` port may be used in the `delegateTo` field of a `DelegatedIBCAuthentication` verification method. Similarly, any channel connected to the `did-app` port may be used in the `serviceEndpoint` field of a `DelegatedIBCService` service.

### Module Callbacks

As mentioned above the module callbacks will be different depending on the channel port. Thus, the DID module will switch on the `portID` and execute the appropriate logic.

```go
// OnChanOpenInit ensures that portID is either `did-auth` or `did-app`,
// ensures that the auth or app version is supported by DID module,
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
    if portID != "did-auth" || portID != "did-app" {
        return error
    }
    if portID == "did-auth" && !SupportedAuthVersion(version) {
        return error
    }
    if portID == "did-app" && !SupportedAppVersion(version) {
        return error
    }
    if order != UNORDERED {
        return error
    }
    return nil
}
```

```go
// OnChanOpenInit ensures that portID is either `did-auth` or `did-app`,
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
    if portID != "did-auth" || portID != "did-app" {
        return error
    }
    if portID == "did-auth" && !SupportedAuthVersion(version) {
        return error
    }
    if portID == "did-app" && !SupportedAppVersion(version) {
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
    switch packet.GetDestPort() {
    case "did-auth":
        appPacket := decodeAppPacket()
        did := appPacket.Data.DID
        document := resolveDID(did)

        // check that the sender of the packet has authority to send on behalf of DID
        // by checking that module is either authenticator of DID document
        // or authenticator of the appropriate delegated service.
        err := checkAuthority(document, packet.GetSourceChannel(), packet.GetSourcePort(), appPacket)
        if err != nil {
            return error
        }

        // forward to requested app module
        channelKeeper.SendPacket(appPacket)
        return nil, successfulAck(), nil
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
    // acknowledgement is default ack provided in channeltypes
    ack := decodeDefaultAck(acknowledgement)
    switch packet.GetSourcePort() {
    case "did-auth":
        if ack.Success() {
            // retrieve the app packet that was intended to be sent by DID
            // stored by DID module
            appPacket := retrieveAppRequest(packet.DID, packet.Sequence)
            channelKeeper.SendPacket(appPacket)
        } else {
            // Delete the service request as the authentication did not succeed
            deleteAppRequest(packet.DID, packet.Sequence)
        }
    default:
        log(ack)
    }
}
```

```go
// OnTimeoutPacket will delete a service request if the DelegatedAuthPacket times out
func OnTimeoutPacket(
    ctx sdk.Context,
    packet channeltypes.Packet,
) (*sdk.Result, error) {
    if packet.GetSourcePort() == "did-auth" {
        // Delete the service request as the authentication did not succeed
        deleteAppRequest(packet.DID, packet.Sequence)
    }
}
```