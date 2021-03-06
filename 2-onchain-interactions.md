# IRC-2 Onchain Interactions

## Outstanding Questions

- Will the onchain service utilize a provided channel wallet? Or will the channel wallet utilize a provided onchain service?
- Could the onchain service and the channel wallet have the same private key? Should they? If so, what's the best way to go about this?
- What is the best way to define wallet --> app --> onchain interaction (i.e. define a `TransactionPending` outbox type for the message router to handle)?
- Should the wallet avoid double storing any data the chain service might store? For example, if there is a challenge in progress, should the wallet not store this data?
- Does the browser wallet run a chain service? Or will the browser rely on an http server running a chain watcher?
- Does retry-on-certain-errors really require any user-provided constructor args? Ideally we'll be able to make reasonable assumptions w/out requiring anything from the user (eg always retry if nonce error & never retry if insufficient funds).
- What are the pros/cons of giving the onchain service a `registerChannel(channelId)` method vs providing a `channelId` in the constructor? Are there any cases where we'd want to start watching arbitrary channel ids after the onchain service has been created?
- Should we have one `OnchainService` per chain or give `OnchainService` a set of providers keyed by chainId?
- Should the `OnchainService` have a concept of a channel wallet? Or should it accept any channel through `registerChannel` and allow the application to handle service <--> wallet communication?
- Are methods like `getEvents` or `getLatestEvent` useful for the channel wallet?

## Overview

Relevant contracts:

- [`AssetHolder`](https://github.com/statechannels/statechannels/blob/master/packages/nitro-protocol/contracts/interfaces/IAssetHolder.sol) (in practice this will be an `EthAssetHolder` or `ERC20AssetHolder` or something else that implements IAssetHolder)
- [`ForceMove`](https://github.com/statechannels/statechannels/blob/master/packages/nitro-protocol/contracts/interfaces/IForceMove.sol)
- [`NitroAdjudicator`](https://github.com/statechannels/statechannels/blob/master/packages/nitro-protocol/contracts/interfaces/NitroAdjudicator.sol) (extends ForceMove)

The onchain service is responsible for:

- Monitoring events for those relevant to a given set of channels (this set of channels is determined at {create,run?}-time)
- Pushing relevant event information into the channel wallet
- Persisting event information to some long-term storage
- Broadcasting onchain actions

When calling contract methods:

- the channel wallet will be the one who holds a key for the channel account, signs channel states, and encodes them into the eth transaction data field.
- the onchain service will be the one who holds a key for an eth account with gas money

The channel wallet & onchain service keys can be the same key if it is funded for gas, but do not have to be.

### Implementation Details

To build iteratively, the implementation should be broken out into three phases:

- **Phase 0**: stateless chain watching and transaction sending
- **Phase 1**: stateful chain watching
- **Phase 2**: lightweight, stateful chain watching with historical chain syncing (could include integration of outside tools, like a [subgraph](https://thegraph.com/docs/introduction#how-the-graph-works) or [dagger](https://github.com/maticnetwork/dagger.js))

### Service Architecture

![alt text](https://github.com/connext/IRCs/blob/master/assets/IRC-2-architecture.png)

### Contract Methods

Throughout the lifecycle of a channel there a few types of onchain interactions that can be taken:

- send funds into a channel via `AssetHolder.deposit(destination,expectedHeld,amount)`
- transfer assets out of a channel via `AssetHolder.transferAll(channelId,allocation)` or `EthAssetHolder.claimAll(channelId,guarantee,allocation)`
- create a new dispute via `NitroAdjudicator.forceMove(...)`
- respond to a dispute via `NitroAdjudicator.respond(...)` or `NitroAdjudicator.checkpoint(...)`
- conclude a dispute via `NitroAdjudicator.conclude(...)`
- Register the results of a dispute on the asset holder via `NitroAdjudicator.pushOutcome(...)`
- helpers: perform 2 of the above actions at once with methods like `pushOutcomeAndTransferAll` or `concludePushOutcomeAndTransferAll`
- others..?

### Contract Events

#### `AssetHolder` emits `AssetTransferred`

This event is emitted at the completion of the `transferAll` or `claimAll` functions in `AssetHolder.sol`. The `transferAll` method is used to disburse funds from ledger or direct channels, while the `claimAll` method will disburse funds to the appropriate guarantor channel. In the process of removing funds from a channel, the state must be finalized between participants onchain using `pushOutcome` (finalization via adjudication) or `conclude` (happy case finalization).

In practice, there are helper methods `pushOutcomeAndTransferAll` as well as `concludePushOutcomeAndTransferAll` that set the onchain state as well as transfer in one onchain transaction.

```typescript
event AssetTransferred(
  bytes32 indexed channelId, // where funds are disbursed from
  bytes32 indexed destination, // external address to wich funds were sent
  uint256 amount // how many unit of given asset were transfered
);
```

#### `AssetHolder` emits `Deposited`

- This event is emitted at the completion of the `deposit` function in both the `ERC20AssetHolder.sol` and the `ETHAssetHolder.sol` (which inherit the `AssetHolder.sol` contract). In the process of funding a channel, the funder must show their counterparty their intent to fund the channel. Both wallets will then expect a corresponding `Deposited` event to be emitted by the contracts before continuing with channel updates.

```typescript
event Deposited(
  bytes32 indexed destination, // channelId to be credited
  uint256 amountDeposited, // the amount that was credited to the channel
  uint256 destinationHoldings // the channel balance post-deposit
);
```

#### `NitroAdjudicator` emits `ChallengeRegistered`

- This event is emitted at the end of the `forceMove` function in `NitroAdjudicator.sol`. It provides all of the information needed for a channel participant to construct a new state when responding to a challenge. This event is emitted both on challenge creation, and challenge response if the participant is trying to play out the channel states onchain rather than resume offchain operations.

```typescript
event ChallengeRegistered(
  bytes32 indexed channelId, // channelId to be adjudicated
  uint48 turnNumRecord, // latest nonce
  uint48 finalizesAt, // unix timestamp when challenge will finalize
  address challenger, // address of participant who registered the challenge
  bool isFinal, // whether challenge state is final
  FixedPart fixedPart, // constant channel properties
  ForceMoveApp.VariablePart[] variableParts, // variable channel properties
  Signature[] sigs, // signatures of channel participants
  uint8[] whoSignedWhat // information to relate signatures to participants
);
```

#### `NitroAdjudicator` emits `ChallengeCleared`

- This event is emitted at the end of the `respond` and `checkpoint` functions in `ForceMove.sol`. When in a challenge, users can choose to resume channel operations offchain by calling either function depending on the signatures on the state. When calling `respond` only need a single turn taker's signature on a higher nonced state is required to clear a challenge, whereas when calling `checkpoint` requires a state signed by all channel participants. Calling either of these functions will set everything except the `turnNum` in the challenge record to empty values.

```typescript
event ChallengeCleared(
  bytes32 indexed channelId, // channelId with existing adjudication
  uint48 turnNumRecord // a nonce supported by channel participants
);
```

#### `NitroAdjudicator` emits `Concluded`

This event is emitted at the end of the `conclude` as well as the `concludePushOutcomeAndTransferAll` methods. Once the challenge has expired, the outcomes must be finalized before funds can be withdrawn, `concludePushOutcomeAndTransferAll` is a helper that executes the entire process in one transaction.

```typescript
event Concluded(
  bytes32 indexed channelId, // channelId with finalized outcomes
);
```

## Interface Proposal

```typescript
import { Address, Bytes32, Uint256 } from "@statechannels/client-api-schema";
import { providers, Wallet } from "ethers";

// Configuraiton for the onchain service, all values have defaults
// if not provided
export type OnchainServiceConfiguration = Partial<{
  transactionAttempts: number; // Maximum number of times a tx is retried
}>;

// This is used instead of the ethers `Transaction` because that type
// requires the nonce and chain ID to be specified, when sometimes those
// arguments are not known at the time of creating a transaction.
export type MinimalTransaction = Pick<
  providers.TransactionRequest,
  "chainId" | "to" | "data" | "value"
>;

// Options (like retry, delay, gas multiple, etc) for the transaction send
// attempt
export type TransactionSubmissionOptions = Partial<{
  maxSendAttempts: number;
}>;

// An injectable service responsible for sending transactions to chain.
// Designed to be used so the OnchainServiceInterface can "set it and forget it"
export interface TransactionSubmissionServiceInterface {

  /**
   * @description Creates a new transaction submission service
   * @param provider RPC Url or a JSON RPC Provider
   * @param wallet Wallet or private key to send transactions from
   */
  constructor(provider: string | providers.JsonRpcProvider, wallet: string | Wallet);
  /**
   * @description Sends a transaction to chain
   * @param minTx Transaction to submit
   * @param options Transaction configuration options
   */
  submitTransaction(
    minTx: MinimalTransaction,
    options?: TransactionSubmissionOptions
  ): Promise<providers.TransactionResponse>;
}

export interface OnchainServiceInterface {
  /**
   * @description Create a new onchain service and begin syncing according to given env
   * @param provider RPC Url or a JSON RPC Provider
   * @param transactionSubmissionService Implements ITransactionSubmissionService
   * @returns A newly started and syncing instance of the onchain service
   *
   * @notice if init code can be run synchronously use constructor, else make it private & use static method (i.e. `create`)
   * @notice Not implemented until v2, may not be implemented at all (mostly included as discussion)
   */
  constructor(
    provider: string | providers.JsonRpcProvider,
    transactionSubmissionService: TransactionSubmissionServiceInterface,
    config: OnchainServiceConfiguration = {}
  );

  /**
   * Removes all listeners from the service, and stops any ongoing operations.
   *
   * @notice May be async, depending on final iterations
   */
  public attachHandler<T extends ContractEvent>(
    assetHolderAddr: Address,
    event: T,
    callback: (event: ChannelEventRecordMap[T]) => void | Promise<void>,
    filter?: (event: ChannelEventRecordMap[T]) => boolean,
    timeout?: number
  ): Evt<ChannelEventRecordMap[T]> | Promise<ChannelEventRecordMap[T]>;
  
  /**
   * Detaches all handlers from the evt instance for the given asset holder
   * and event (if provided)
   * @param assetHolderAddr Contract address of Asset Holder
   * @param event Event to remove handlers from
   *
   * @notice This may not be strictly necessary, but is useful for testing
   */
  public detachAllHandlers(assetHolderAddr: Address, event?: ContractEvent): void;

  /**
   * @description Registers a channel for the onchain service to watch for.
   * @param channelId Unique channel identifier
   * @param assetHolders Asset holder contract addresses
   */
  public registerChannel(
    channelId: Bytes32,
    assetHolders: AssetHolderAddress[]
  ): Promise<void>;

  /**
   * @description Submits a transaction to chain
   * @param channelId Unique channel identifier
   * @param tx Minimum transaction to send
   * @returns A transaction response
   *
   * @notice Why do we need to provie a channelId? (for storage purposes, would be useful to know)
   */
  public submitTransaction(
    channelId: Bytes32,
    tx: MinimumTransaction
  ): Promise<providers.TransactionResponse>;

  /**
   * @description Associates a channel wallet with the service, allowing the onchain service to
   * execute the required callback following an event for a registered channel
   * @param wallet The channel wallet instance
   * @returns void
   *
   * @notice Should we remove in V1? (See questions section)
   */
  public attachChannelWallet(wallet: ChannelWallet): void

  /**
   * @description Retrieves the latest event of a given type for the provided channel
   * @param channelId Unique channel identifier
   * @returns The latest chain event if found, undefined otherwise
   *
   */
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
