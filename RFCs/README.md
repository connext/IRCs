# RFCs and Specifications for I3
**IRC**s -- **I**ndra **R**equests for **C**omment.

### Contributing
New IRCs must be submitted via PR by taking an issue from the TODOs below.

Each IRC has a `finalization date` within which decisions regarding the IRC MUST be finalized and the IRC PR, closed. Before this period, IRCs should be commented on in their respective PRs.

TODOs:
- [ ] **IRC1** `Finalization date: Tues Aug 25th` -- Specification of StateChannels `server-wallet` interface. Includes:
    - Spec of a `mock-wallet` which can be used for unit testing. 
    - Architecture diagram of StateChannels wallet internals.
    
- [ ] **IRC2** `Finalization date: Tues Aug 25th` -- System architecture of new node infrastructure. Includes:
    - Architecture diagram of the new node
    - Key decisions around supported interfaces. REST? gRPC? Both? Something else?
    - Finalized naming conventions and terminology for each piece. Nodes/clients? Routing/Non-routing nodes?
    - Interfaces that have been defined vs. are yet to be defined? What can/should we reuse?
