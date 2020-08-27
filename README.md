# RFCs and Specifications for Indra3
**IRC**s -- **I**ndra **R**equests for **C**omment.

The files in this repository detail the protocol layers currently under development for the next iteration of Connext's state channel network. These specifications are a work in progress.

We actively encourage pull requests and are looking to get as much community feedback as possible. Before submitting or commenting on a PR, please review [Contributing](https://github.com/connext/IRCs/blob/master/README.md#contributing) below for some basic guidelines.

## Intro

Connext is a state channel network - a p2p overlay network that sits on top of (and across) EVM-compatible blockchains and allows for instant, near-free mutations to blockchain state without compromising the trust-minimization and decentralization qualities of the underlying chain.

This proposed iteration of Connext extends [State Channels](https://statechannels.org), a minimal, generalized framework for developing state channel applications.

## Table of Contents
### Core
1. [IRC-0: Base Protocol Background](https://github.com/connext/IRCs/blob/master/0-base-protocol-background.md) - Background and JSON RPC Specification of StateChannels.
2. [IRC-1: System Architecture](https://github.com/connext/IRCs/blob/master/1-system-architecture.md) - Architecture of Indra node and overview of modules.

## Contributing
New IRCs must be submitted via PR by taking an issue from the TODOs below.

TODOs:
- **IRC-2** -- Specification of Message-Router and messaging infra. Includes:
    - Key decisions around transport
    - Message-router interface
    - Auth/encryption strategy
    - Connection to lower level wallet interface
    - Specification of message retries and failure cases
- **IRC-3** -- Specification of Chain Watching/Transaction infra. Includes:
    - Interface to StateChannels
    - Discussion of environment agnosticism
    - Historical state vs current state of chain
    - Working with existing infra that's out there? (Graph, Any.sender)
