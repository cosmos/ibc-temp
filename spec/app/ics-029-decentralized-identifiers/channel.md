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
