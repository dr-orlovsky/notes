# API design considerations

## Background

There multiple different types of API that can be used by the software stack
related to LNP/BP. Here we analyze criteria to choose the proper API
technologies and serialization standards for different cases.

In general, software might require API for:
* **Interprocess communications** (IPC), including those between daemons, their
  instances, or used in microservice architectures on the same machine or in
  network-connected docker containers behind DMZ.
* **Non-web-based client-server** interactions crossing DMZ, following either
  request/reply or subscribe/publish patterns.
* **REST Web-based APIs** for requesting resource-based data (i.e. with clear
  data hierarchical structure)
* **Real-time or transactional web-based APIs**, including requests for remote
  procedures (RPC) over web or bidirectional/realtime (RT) client-server
  communications using Websockets.

API type              | Sample use cases | Typical scenarios
----------------------|------------------|-----------------------
IPC                   | c-lightning IPC  | Microservice IPC for servers and daemons
Non-web client-server | electrum protocol; bitcoind RPC | High-throughput or non-custodial solutions
Web-based REST        | esplora          | Blockchain explorers
Web-based RPC/RT      | many web apps    | Wallets

Today, many different API description languages, serialization formats and
transport layers exist that may be used in the mentioned scenarios. However, in
most of the cases the choice of the particular formats are nearly arbitrary or
related to historical reasons. Here I'd like to systematize criteria for API
technic selection in LNP/BP for future apps that may allow to avoid many bad
practices of the past.

## Overview

### API components

The classical API consists of three main components:
1. Data serialization format, allowing all parties involved in communication
   read/write data with the same deterministic result. Usually classified to be
   binary or textual basing on human readibility/ASCII character set. The most
   common formats are:
    * ASCII-based/text/human readable:
        - XML (XML Schema)
        - JSON (JSON Schema)
        - YAML (YAML Schema)
    * Binary serialisation
        - BSON
        - Protocol buffers
        - ASN.1
        - RPC framework-specific (like used in ZMQ, Apache Thrift)
        - Custom/vendor-specific (Bitcoin core and others)
   Many serialization formats have a schema- (mostly for human-readable) or
   DSL-based definition of possible values used by a particular application/API,
   which may be used for an automatic code generation and/or data packet
   validation.
2. API per se, specifying available resources or procedures which may be invoked
   via IPC/network communication. Thus, API may fall into two classes:
   * RPC (remote procedure call), where each API call consists of the invoked
     procedure name and a list of it's arguments - very much alike procedural
     programming languages. Server-side components with RPC paradigm usually
     have their state.
   * REST (representational state transfer), used to call ACID-based methods
     on a well-defined hierarchical graph of resources
   * Custom/non-standard approaches, like GraphQL
3. Transport-layer protocol, defining the means of transporting information
   about API calls and associated data over the underlying network topology:
   * POSIX sockets
   * POSIX IPC
   * TCP/IP
   * UDP/IP
   * HTTP (pure or over TLS/SSL)
   * Websockets (pure or over TLS/SSL)
   * Tor/SOCKS


Many existing API automation frameworks (see below) cover more than a single
API component.


### API Protocols and Frameworks

Here we provide information only about modern and most recently used frameworks:

Framework name / protocol family | Layers   | Transport protocol requirements    | Best suited/designed for
---------------------------------|----------|------------------------------------|----------------
Apache Thrift  | 1 (many), 2 (RPC), 3 (custom) | HTTP(s), TCP                    | Microservice architectures (only Req/Rep however)
GraphQL        | 1 (JSON), 2 (custom)       | HTTP(s)                            | Complex data-centric web applications with non-hierarchical data graphs
gRPC/Protobuf  | 1 (binary/custom), 2 (RPC) | HTTP(s), TCP, ?                    | Microservice architectures (only Req/Rep however)
JSON-RPC       | 1 (JSON), 2 (RPC)          | HTTP                               | Legacy/insecure
OpenAPI        | 1 (JSON), 2 (REST)         | HTTP(s)                            | REST web applications
SOAP/WSDL      | 1 (XML), 2 (RPC)           | HTTP                               | Enterprise system bus-centered enterprise architectures
WAMP           | 1 (JSON or other), 2 (RPC) | Websockets, TCP, POSIX             | Real-time web apps, socket-based apps
XML-RPC        | 1 (XML), 2 (RPC)           | HTTP                               | Legacy/insecure
ZeroMQ         | 1 (binary/custom), 2 (RPC) | POSIX sockets, POSIX IPC, TCP, USD | High throughput, Pub/Subs, IPCs, Microservice architectures


## IPC for Microservices

The requirements for this are:
* Compact binary data serialization format
* Support for custom serialization (i.e. consensus-based for Bitcoin-related
  data structures)
* No third-party code generation tools (safety for consensus-critical data)
* High throughput transport
* Support for all types of IPC sockets including Tor
* Ability to use encryption at transport layer
* Support of Request-Reply (RPC) and Publish-Subscribe patterns
* Well suited for serialization of hashes, public keys etc.

Much less important for the protocols:
* Web compatibility
* Human readability

ZeroMQ seems to be a tool of choice for the transport layer, which have to be
combined with custom RPC API DSL and serialization protocol.

## Client-server (non-web)

ZeroMQ seems to be the tool of the choice here as well

## Web-based REST

OpenAPI seems to be the tool of the choice.

## Web-based RPC

WAMP seems to be the tool of the choice for apps that require live updates
(Websockets).

Another alternative to consider is GraphQL, however it should be noted that id
usually has a poor performance and is not suited for Websocket apps.

## End notes

Protocol buffers or Apache Thrift serialization can't be used in all of the cases due to:
* A lot of code generation
* No support for hashes or public keys
