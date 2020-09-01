# IRC-3 Message Router

## Overview

The `message-router` component is a major piece of architecture in the isomorphic non-routing node. Its responsibility is to translate messages from the external messaging layer and interact with the `wallet` through a JSON-RPC interface, and relay protocol messages to a counterparty.

## Architecture

The `message-router` is isomorphic, so it is unopinionated about the type of messaging layer it sits under. The `message-router` interface is designed to be interacted with through a controller which can be a REST API interface, gRPC, a browser interface, or some other type of RPC interface. It can also be consumed by a higher-level service which contains app context and business logic.

The `message-router` contains two important dependencies. One is the interface to the wallet, the `WalletRpcService`, and one is the `MessagingService` which wraps the transport-specific channel protocol messaging. The `message-router` must also handle creating the "inbox" subscription that receives messages from a counterparty and pushes them into the `wallet` through the `WalletRpcService`.

The `message-router` will handle cases where requests either go to chain or to a counterparty. In these cases, we follow a pattern where the function call itself returns as early as possible and passes back a Promise that resolves when the long-running operation is completed.

## Interface

```typescript
import {
  ChannelResult,
  Participant,
  ChannelId,
  Address,
  UpdateChannelParams,
} from "@statechannels/client-api-schema";
import { BigNumber } from "ethers";

type Completed<T> = {
  completed(): Promise<T>;
};

export type DepositParams = {
  channelId: ChannelId;
  amount: BigNumber;
  assetId: Address;
};

export type WithdrawParams = DepositParams;

export interface IMessageRouter {
  createChannel(
    counterparty: Participant
  ): Completed<ChannelResult> & ChannelResult;
  deposit(params: DepositParams): Completed<ChannelResult> & { txHash: string };
  updateChannel(params: UpdateChannelParams): ChannelResult;
  withdraw(params: WithdrawParams): ChannelResult & { txHash: string };
}
```
