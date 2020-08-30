# IRC-0 Background on StateChannels Protocol

## Overview

State channels is a core protocol "engine" (they call it a `wallet`) that validates and signs channel updates.

It is similar to our existing CFCore module in the following ways:

1. The core "application" lifecycle of `propose`--> `install`--> `takeAction` --> `uninstall` is retained.
2. Data flows through an external transport layer (for inbound messages from a counterparty) or JSON rpc interface (for user-initiated inputs), gets validated by the protocol against previous state, and then gets stored in store.
3. The output of state transitions is a set of commitments that can be put onchain in the event of a dispute.

It is different in the following ways:

1. The "application" lifecycle is named `create`--> `join`--> `update`--> `close`. The same cycle is used both for "subchannels"/virtual channels (i.e. apps), **and** for the ledger channel. This dramatically simplifies the interfaces.
2. The core protocol handles deposit/withdraw natively. //TODO How do channel topups and partial withdraws work?
3. There are no more multisig contracts. Instead there's a global `assetHolder` contract per asset. //TODO How do we deposit directly to an address? How do we allow counterparty to pay for withdrawal?
4. The StateChannels wallet does not handle message dispatching and responding directly. Instead, it uses a `pushMessage` pattern whereby the implementer pushes updates into the wallet (it also emits events to the application layer using `onNotification`). This dramatically simplifies adding middleware for encryption, message retries, caching, etc.
5. The protocol extends turn based updating from being only an "app"-level feature to instead be throughout the protocol. This removes the need for distributed locks and, while it's still possible for states to get out of sync, makes recovery _very_ easy.
6. The application solidity code no longer defines a state machine but instead a simpler interface which just ensures that a given transition is valid. This means that calculating state transitions themselves is outside the scope of the wallet.
7. The wallet is more tightly coupled with the store implementation and cannot easily be abstracted into a general interface with pluggable stores. This is because wallets are making use of transactions within the store itself (i.e. database transactions in the server-wallet) to serialize channel operations.
8. For now, the wallet does not allow for an externally passed in signer. This is something that can be changed, however.

## Terminology

Some terms and naming conventions have been changed in the new protocol:

- **Sub/virtual/ledger channel**: A ledger channel refers to a channel whose funding is in an onchain address. A "sub" channel is what we currently call an "app". A virtual channel is a sub channel constructed over multiple ledger channels through one or many intermediaries.
- Distinction between apps/channels: In StateChannels, there is no difference between ledger vs sub/virtual channels. This means `createChannel` refers to _both_ creating a channel onchain and to installing an app.
- **ChannelId**: Corresponds to existing `multisigAddress` for a ledger channnel or existing `appIdentityHash` for a sub/virtual channel. Derived using `participants[]`, `chainId`, `nonce`, `timeout`, ``appDefinition` //TODO: check
- **ParticipantId**: Unique identifier for a given participant. For now, this is the equivalent of `signerAddress`.

## Architecture

Dependency graph of StateChannels modules:

![alt text](https://github.com/connext/IRCs/blob/master/assets/IRC-0-SC-dependency.png?raw=true)

Call flow:

```
      +--------------+                 +-----------------+
      | Payer Wallet |                 | Receiver Wallet |
      +-+-+----+-+---+                 +-+-+----+-+------+
        ^ |    ^ |                       ^ |    ^ |
        | |    | |                       | |    | |
update  0 1    8 9 pushMessage    update 5 6    3 4 pushMessage
Channel | |    | |               Channel | |    | |
        | v    | v                       | v    | v
      +-+-+----+-+---+                 +-+-+----+-+------+
      |              +-------2-------->+                 |
      | Payer Client |  POST /inbox    | Receiver Client |
      |              +<------7---------+                 |
      +--------------+                 +-----------------+
```

Where clients are implemented by us.

## JSON RPC Interface

A WIP specification of the JSON RPC interface [can be found here](https://github.com/connext/statechannels/blob/client-api-docs/packages/docs-website/docs/protocol-docs/client-specification/json-rpc-api.md). We can eventually move the WIP spec into this doc.

## Missing Features

Some core features are still being worked on in the `server-wallet` and will need to be completed before a Connext node built on StateChannels could be properly run and tested. Aside from finalizing the above JSON RPC schema, these are (in rough order of priority):

- [ ] Support for ledger channel funding
- [ ] Depositing into the channel
- [ ] Defuning a channel on close
- [ ] Top-ups + partial withdrawals
- [ ] Support for virtual channels
- [ ] Single ERC20 support
- [ ] Deposit directly to an address instead of calling the contract method
- [ ] Multiple ERC20 support
- [ ] ChannelSigner support
- [ ] ChainId cleanup (right now, all chainIds are hardcoded to `0x01`)
- [ ] Protocol version number in messages (and corresponding error handling)
- [ ] Store version number
- [ ] Support for backup / restore from state
- [ ] Migration path coordination from existing stack to StateChannels
