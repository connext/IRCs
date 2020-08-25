# IRC-1 Indra System Architecture

![alt text](https://github.com/connext/IRCs/blob/01-system-architecture/assets/IRC-1-architecture.png?raw=true)

## Moving to a Node-to-Node System

In the last iteration of Indra, we had separate constructs for `clients` and `nodes`. Clients were non-routing implementations of Connext that could be run in any environment and made no assumptions about liveness. Nodes (equiv. "hubs") were always-online routing implementations.

One important client abstraction that emerged from users trying to integrate Connext was the [rest-api-client](https://github.com/connext/rest-api-client), a client implementationt that is designed to run in a server environment and contains some basic logic for key/channel management. In the process of developing this client wrapper, we observed that it could *and should* actually behave exactly like the node in most cases (except for routing/collateralizing transactions).

This time around, we've decided to maximize code reuse by treating the concept of the `rest-api-client` as the node itself. In the above diagram, note that there are no `client`s, just `nodes` of either a `routing` or `non-routing` variety. A (non-routing) `node` (todays equivalent of a client running on a server) contains all of the functionality needed for a user to utilize one or many channels in a backend env manually (i.e. by encoding their own business logic). A `routing node` is a `node` that consumes the `node` gRPC interface and "automates" the process of forwarding/collateralizing transactions.

### Key Benefits
1. **Code reuse:** with this structure, separating several different clients and a node implementation becomes unnecessary. 
2. **Testability** a single set of tests against the ` node` interface would cover the majority of code surface area. Setting up integration tests could happen as soon as the core `node` code is done.
3. **Time to launch**: for backend-only usecases, this approach would yield a working implementation in a small fraction of the time that it would take otherwise.
4. **Easier path to multihop**: initially, we'll restrict the ability for `routing node`s to connect to one another to keep surface area low. Adding support for multihop, however, will simply comprise an update to the `routing-service` and loosening this restriction.

Note: this architecture implies that we will not add support for React-Native and Browser builds until after the core build is complete. However, once a fully working implementation of the `routing node` is stable, developing a browser/rn implementation against it will not be too difficult.

## Modules/Services Overview

### ChannelSigner and Keyring
Similar to how both our current `indra-node` and `rest-api-client` work, the `node` would store keys in a docker secret in the env and be responsible for managing them safely. It would also create a `ChannelSigner` with the same interface as what exists in the current implementation that would be injected into the StateChannels `server-wallet` for signing.

### Message-Router
//TODO Should this be called `networking-service`?

Message-router would be responsible for the following tasks:
1. Pushing messages into the `server-wallet` (and getting corresponding outputs) using `pushMessage`
2. Dispatching those messages to a recipient via some transport layer. For now, we will use [nats](https://nats.io) like in the existing implementation. Also note that we make some assumptions here about message addressing -- with nats, discovery and addressing are built in, but if we move to another transport eventually, we'd need to build these in.
3. Authentication/encryption tasks related to the above.
4. Listening to the `onNotification` event emitter and bubbling those events up to higher level modules. //TODO is this correct?
5. Timing message responses and retrying if there is no response within the timeout.
6. Filtering for duplicate messages -- the `server-wallet` may already handle this? //TODO

### Default-apps
There are a set of default applications and actions we want to include as part of the core node stack. These apps represent a default set of utilities that should cover some or all of the basic needs of *most* implementers. Examples include simple transfers, async transfers, htlcs, top-ups, partial withdraws.

### REST && gRPC Node APIs
One major problem with the current structure of the `indra-node` is that there is no way to interact directly with a channel. Even if we expose direct access to a node's `CFCore` modules, modifying a channel or even querying channel data requires in-depth knowledge of the Counterfactual protocol and manually encoded params that are generally entirely abstracted from the user. This makes it really difficult to do basic tasks like:
1. Query channel data from the node.
2. Build new node services/business logic (that doesn't rely on clients to initiate interactions).
3. Manually take actions in a specific channel for debugging or recollateralization processes.

The `node` API acts as a way to interact with one or many channels associated with the `node`s key(s). It should look very similar to the existing client API but with some more lower level functionality exposed. Doing this means that the `routing` portion of a `routing node` can sit entirely independently of the core implementation, and consume the exact same interface that the user would consume already. It also means we can iterate on rebalancing logic or even build entirely new tools like a node CLI without modifying any node code.

### Routing-Service
//TODO Should this be called `forwarding-service`?

The `routing-service` is responsible for turning a simple node (effectively what a client is today) into a `routing node` (indra-node or hub today). The `routing-service` listens to events bubbled up from `server-wallet`, and then takes actions on one or many channels to forward hashlocked transfers OR set up virtual channels.

In the current implementation, this functionality is implemented in a combination of `transfer.service.ts`, `appActions.service.ts` and `appRegistry.service.ts`. Because we haven't implemented VCs, however, the process for adding support for new apps is still very high touch and modifies some of the core transfer flow.

In the next iteration, we should utilize virtual channels for **all** current `requireOnline` flows. For now, the only transfer type where a recipient may be offline (which would not work with a virtual channel), would be the async transfer. We should limit hashlock-forwarding-based flows to **only** be for the async transfer for now.

// TODO what needs to happen to support virtual channels?

### Rebalancing-Service
//TODO