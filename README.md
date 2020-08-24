# RFCs and Specifications for Indra3
**IRC**s -- **I**ndra **R**equests for **C**omment.

The files in this repository detail the protocol layers currently under development for the next iteration of Connext's state channel network. These specifications are a work in progress.

We actively encourage pull requests and are looking to get as much community feedback as possible. Before submitting or commenting on a PR, please review [Contributing](https://github.com/connext/IRCs/blob/master/README.md#contributing) below for some basic guidelines.

## Intro

Connext is a state channel network - a p2p overlay network that sits on top of (and across) EVM-compatible blockchains and allows for instant, near-free mutations to blockchain state without compromising the trust-minimization and decentralization qualities of the underlying chain.

This proposed iteration of Connext extends [State Channels](https://statechannels.org), a minimal, generalized framework for developing state channel applications.

## Table of Contents
### Core
1. [IRC-0: Base Protocol Background](https://github.com/connext/IRCs/blob/01-base-protocol-background/0-base-protocol-background.md) - Background and JSON RPC Specification of StateChannels.

## Contributing
New IRCs must be submitted via PR by taking an issue from the TODOs below.

Each IRC has a `finalization date` within which decisions regarding the IRC MUST be finalized and the IRC PR, closed. Before this period, IRCs should be commented on in their respective PRs.

TODOs:
- [ ] **IRC-0** `Finalization date: Tues Aug 25th` -- Background specification of StateChannels wallet interface. Includes:
    - JSON RPC interface for wallet
    - Spec of a `mock-wallet` which can be used for unit testing. 
    - Architecture diagram of StateChannels wallet internals.
    
- [ ] **IRC-1** `Finalization date: Tues Aug 25th` -- System architecture of new node infrastructure. Includes:
    - Architecture diagram of the new node
    - Key decisions around supported interfaces. REST? gRPC? Both? Something else?
    - Finalized naming conventions and terminology for each piece. Nodes/clients? Routing/Non-routing nodes?
    - Interfaces that have been defined vs. are yet to be defined? What can/should we reuse?
