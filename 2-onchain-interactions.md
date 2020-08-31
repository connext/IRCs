# IRC-2 Onchain Interactions

## Overview

Throughout the lifecycle of a channel there two major types of onchain interactions required:

- monitoring + storing contract events
- sending transactions to chain (during adjudication, funding, withdrawing)

Both of these functions are handled by an onchain service, which the channel wallet has access to.

## Protocol Events Background

The statechannels protocol uses the following funding events defined in [`IAssetHolder.sol`](https://github.com/statechannels/statechannels/blob/master/packages/nitro-protocol/contracts/interfaces/IAssetHolder.sol):

- **AssetTransferred**: emitted when transferring funds out of the channel
- **Deposited**: emitted when transferring funds into the channel

as well as the following adjudication events defined in [`IForceMove.sol`](https://github.com/statechannels/statechannels/blob/master/packages/nitro-protocol/contracts/interfaces/IForceMove.sol)

- **ChallengeRegistered**: emitted when challenge is created or advanced to a higher nonce
- **ChallengeCleared**: emitted when a challenge response or checkpoint occurs, and channel operations are resumed offchain
- **Concluded**: emitted when outcomes are finalized

### AssetTransferred

The `AssetTransferred` event is defined as:

```typescript
event AssetTransferred(
  // channelId funds were disbursed from
  bytes32 indexed channelId,
  // external address funds were sent to
  bytes32 indexed destination,
  // amount disbursed
  uint256 amount
);
```

This event is emitted at the completion of the `transferAll` or `claimAll` functions in the `AssetHolder.sol`. The `transferAll` method is used to disburse funds from ledger or direct channels, while the `claimAll` method will disburse funds to the appropriate guarantor channel. In the process of removing funds from a channel, the state must be finalized between participants onchain using `pushOutcome` (finalization via adjudication) or `conclude` (happy case finalization).

In practice, there are helper methods `pushOutcomeAndTransferAll` as well as `concludePushOutcomeAndTransferAll` that set the onchain state as well as transfer in one onchain transaction.

### Deposited

The `Deposited` event is defined as:

```typescript
event Deposited(
  // channelId to be credited
  bytes32 indexed destination,
  // the amount that was credited to the channel
  uint256 amountDeposited,
  // the channel balance post-deposit
  uint256 destinationHoldings
);
```

This event is emitted at the completion of the `deposit` function in both the `ERC20AssetHolder.sol` and the `ETHAssetHolder.sol` (which inherit the `AssetHolder.sol` contract). In the process of funding a channel, the funder must show their counterparty their intent to fund the channel. Both wallets will then expect a corresponding `Deposited` event to be emitted by the contracts before continuing with channel updates.

### ChallengeRegistered

The `ChallengeRegistered` event is defined as:

```typescript
event ChallengeRegistered(
  // channelId to be adjudicated
  bytes32 indexed channelId,
  // latest nonce
  uint48 turnNumRecord,
  // unix timestamp when challenge will finalize
  uint48 finalizesAt,
  // address of participant who registered the challenge
  address challenger,
  // whether challenge state is final
  bool isFinal,
  // constant channel properties
  FixedPart fixedPart,
  // variable channel properties
  ForceMoveApp.VariablePart[] variableParts,
  // signatures of channel participants
  Signature[] sigs,
  // information to relate signatures to participants
  uint8[] whoSignedWhat
);
```

This event is emitted at the end of the `foceMove` function in `ForceMove.sol`. It provides all of the information needed for a channel participant to construct a new state when responding to a challenge. This event is emitted both on challenge creation, and challenge response if the participant is trying to play out the channel states onchain rather than resume offchain operations.

### ChallengeCleared

The `ChallengeCleared` event is defined as:

```typescript
event ChallengeCleared(
  // channelId with existing adjudication
  bytes32 indexed channelId,
  // a nonce supported by channel participants
  uint48 turnNumRecord
);
```

This event is emitted at the end of the `respond` and `checkpoint` functions in `ForceMove.sol`. When in a challenge, users can choose to resume channel operations offchain by calling either function depending on the signatures on the state. When calling `respond` only need a single turn taker's signature on a higher nonced state is required to clear a challenge, whereas when calling `checkpoint` requires a state signed by all channel participants. Calling either of these functions will set everything except the `turnNum` in the challenge record to empty values.

### Concluded

The `Concluded` event is defined as:

```typescript
event Concluded(
  // channelId with finalized outcomes
  bytes32 indexed channelId,
);
```

This event is emitted at the end of the `conclude` as well as the `concludePushOutcomeAndTransferAll` methods. Once the challenge has expired, the outcomes must be finalized before funds can be withdrawn, `concludePushOutcomeAndTransferAll` is a helper that executes the entire process in one transaction.

## Design Requirements

The onchain service must:

- monitor the chain for channel events
- push event information to the channel wallet for relevant channels
- store information related to relevant channel events
- send transactions with sent to the service from the channel wallet

### Implementation Details

To build iteratively, the implementation should be broken out into three phases:

- **Phase 0**: stateless chain watching and transaction sending
- **Phase 1**: stateful chain watching
- **Phase 2**: lightweight, stateful chain watching with historical chain syncing (could include integration of outside tools, like a [subgraph](https://thegraph.com/docs/introduction#how-the-graph-works) or [dagger](https://github.com/maticnetwork/dagger.js))

### Service Architecture

![alt text](https://github.com/connext/IRCs/blob/master/assets/IRC-2-architecture.png?raw=true)

### Interface

```typescript
import { providers, Wallet } from "ethers";

type OnchainServiceEnvironment = {
  provider: string | providers.JsonRpcProvider;
  wallet: string | Wallet; // Private key or wallet to send tx
  maximumRetries?: number; // Maximum number of times a tx is retried
};

type MinimumTransaction = {
  to: Address;
  value: Uint256;
  data: string; // calldata
};

type TransactionResult = providers.TransactionResponse & {
  // TODO: this type of callback is useful if you need to wait not
  // only for the transaction but also for some other transaction
  // related asynchronous behavior (i.e. writing event data to storage)
  completed: () => providers.TransactionReceipt & ContractEvent;
};

type ChainEventName =
  | "fundsDeposited"
  | "fundsWithdrawn"
  | "transactionMined"
  | "forceMoveDetected"
  | "checkpointDetected"
  | "concludeDetected"
  | "respondDetected";

type ChainEvent = {
  channelId: Bytes32;
  event: {
    type: ChainEventName;
    // Example data type
    // https://github.com/statechannels/statechannels/blob/master/packages/nitro-protocol/src/contract/asset-holder.ts#L5
    data: ContractEvent;
  };
};

export interface IOnchainService {
  constructor(config: OnchainServiceEnvironment);

  /**
   * @description Static method to create and begin syncing an onchain service
   * @param config The environment for the service
   * @returns A newly started and syncing instance of the onchain service
   *
   * @notice Not implemented until v2, may not be implemented at all (mostly included as discussion)
   */
  public static start(
    config: OnchainServiceEnvironment
  ): Promise<IOnchainService>;

  /**
   * @description Stops the onchain service from monitoring the chain
   * @returns A boolean indicating success
   *
   * @notice Not implemented until v2, depends on how historical syncing is done
   */
  // TODO: see below for note about booleans
  public stop(): Promise<boolean>;

  /**
   * @description Connects the wallet to the given provider
   * @param provider The provider to connect to
   * @returns The updated service
   */
  public connect(
    provider: string | providers.JsonRpcProvider
  ): Promise<IOnchainService>;

  /**
   * @description Registers a channel for the onchain service to watch for.
   * @param channelId Unique channel identifier
   * @param assetHolders Asset holder contract addresses
   * @returns boolean value indicating if registration was successful
   */
  // TODO: booleans for success are rarely very informative, since you get little
  // insight into why the failure happened. I would suggest returning a string value
  // instead
  public registerChannel(
    channelId: Bytes32,
    assetHolders: AssetHolderAddress[]
  ): Promise<boolean>;

  /**
   * @description Submits a transaction to chain
   * @param tx Minimum transaction to send
   * @param completionEvent An event you expect to be emitted when the transaction can be
   * considered "completed" from the POV of the chain wallet (i.e. all data stored, see TODO
   * above if this is even necessary)
   * @returns A transaction result
   */
  public submitTransaction(
    tx: MinimumTransaction,
    reason: 'challenge' | 'deposit' | 'withdraw'
  ): Promise<TransactionResult>;

  /**
   * @description Retrieves the latest event of a given type for the provided channel
   * @param channelId Unique channel identifier
   * @returns The latest chain event if found, undefined otherwise
   */
  // TODO: will this be useful for the channel wallet?
  public getLatestEvent(
    channelId: Bytes32,
    event: ChainEvent
  ): Promise<ChainEvent | undefined>;

  /**
   * @description Retrieves all event records for a given channel
   * @param channelId Unique channel identifier
   * @param event If provided, only returns events of a specific type for the channel (optional)
   * @returns All channel events
   *
   * @notice Not implemented in v0
   */
  public getEvents(
    channelId: Bytes32,
    event?: ChainEvent
  ): Promise<ChainEvent[]>;
}
```
