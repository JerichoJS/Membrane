![Membrane](./Assets/Membrane-banner.svg)</br>
![](https://api.codiga.io/project/33828/score/svg)
![](https://img.shields.io/github/license/JerichoJS/membrane?color=blue&label=License)
![](https://img.shields.io/github/languages/code-size/JerichoJS/membrane?color=%20%23d0a011%20&label=Raw%20Code%20Size)
[![](https://img.shields.io/website?down_color=%23D0342C&down_message=Offline&label=Website%20Status&up_color=%23e8daef&up_message=Operational&url=https%3A%2F%2Fmembra.ne)](https://membra.ne)
</br>
A robust, minimal-server-interaction API for peer routing in the browser

## What  is this?
The official membrane framework, specifically `index.js` within `/lib`, capitalizes upon the `RTCPeerConnection` API's inherent agnosticism regarding signaling. That you could just as well communicate the `ICE` data necessary for establishing a peer connection through smoke signals or quantum teleportation (*if only*), as through a more conventional signalling server means very little constrains us to this often-unreliable, centralized approach. Fundamentally, Membrane allows us to create unencumbered, living peer networks, so that, after just a single centralized signal, we can enter an ultra-fault-tolerant network of thousands of highly interconnected nodes, routing by peer-borne signals to any node we desire. Once we are "in," we are so for good. No matter what happens to our original signaling server, the network will remain untouched.

However, this approach is far from perfect. This very hyper-decentralization that makes us so indestructible--our greatest boon--in turn becomes our worst enemy. Without a singular, trusted register to authenticate peers, spoofing, posing, and generally manipulating the structure of the network become elementary. I plan on soon implementing a direct, key-derivation-based codeword authentication system, but even this requires a centralized medium of initial exchange.

In brief, this tool is robustly functional at enabling anonymous, homogeneous, untrusted data exchange across a network, but very poor at most else.

## Documentation
In the following section I have compiled a set of hopefully-only-slightly-pedantic descriptions for `lib`'s key functionalities. Additionally, it is advised you have a general understanding of the WebRTC API, as it is employed heavily throughout this project.
- ### Aliases
    Aliases are one of the most fundamental properties in this library, defining the unique identities of peers, both formally internally, in the form of `hiddenAliases` and informally to users, in the form of `publicAliases`. `hiddenAliases` are primarily used in transactions and routing between peers and across the network, whereas `publicAliases` are merely a cosmetic abstraction only ever seen by users. The two are correlated by the global-scope objects `hiddenAliasLookup` for finding corresponding `publicAliases` and `publicAliasLookup` for the converse.
- ### Config
  - #### Loading custom configurations:
    All global config is loaded into the `CONFIG` object from initialization, with the contents of `defaultConfig` acting as a baseline template, which its own `constants\configLoadFunction` extends, providing a set of values to substitute, formatted as follows:
    ```
    {
        rootType.subtype.(...).preferenceName : value
        ...
    }
    ```
    This function is run every time the script is initialized, invoked by `loadConfig`, and may be left undefined or simply return `{}` to use the exact `defaultConfig`. If a path is provided which has not been pre-defined by `defaultConfig`, its value is discarded.
  - #### communication.packageArgs 
    An array of arguments which must/may be included in packages sent over `RTCDataChannel.prototype.standardSend`. This is used in determining the validity of packages received from peers. Each entry is structured as `{ required : [...], optional: [...] }`. Optional is left open-ended by default, and may be constrained to only the arguments specified, so that any extraneity causes an error by including the value "`!*`."
  - #### communication.routeAcceptHeuristic
    A discriminator function provided a single argument, `routePackage`, representing all non-type data provided in the initial request. If this function is `async` it will be `await`ed  by default, allowing asynchronous user interaction. Ultimately, this function returns a boolean value, representing whether a route is to be established or not.
- ### Prototype overrides
    In total, this module extends exactly two prototypes, in both cases to add a specific formatting micro-protocol, and both times with unusual, unambiguous names in order to prevent potential future conflicts. **Note**: courses of action delineated here to be taken by the server are specific to the signaling server provided in `/src/server` but can certainly differ, or even be cut altogether to reduce server-node interaction, in a custom server implementation.
    - #### WebSocket.prototype.crudeSend
        This function accepts a mandatory first argument, `type`, and an optional second `typeArgs`--an object containing data relevant to the specific type. This data is then bundled appropriately and sent to the server. We define the following acceptable types:
        - heartbeat - Generates an empty message to indicate a node is still living, as websocket does not implement any ping-pong functionality natively.
        - reportNode - Indicates to the server that a particular node provided an invalid SDP package to a node freshly entering the network. The server will increase this node's routing weight (making it less probable it will route to it again), and provide a new route to the caller. If the node receives invalid data three times in a row, it will throw an error and stop attempting.
        - returnSDP - Returns SDP generated in response to an SDP request from the server.
        - ignoreSDPRequest - Like reportNode but for nodes of which SDP was requested; if a node receives an SDP request which itself is improperly formatted, it will call this. The server will modestly penalize the node reporting the error, and send an `["ERROR"]` package to the initial requester.
    - #### RTCDatachannel.prototype.standardSend
      Simmilarly to crudeSend, this function accepts at minimum one parameter, or two optionally. However, no checks are performed to ensure packages conform to formatting standards, and therefore must be done before calling this function. For a full list of possible inputs, see [CONFIG.communication.packageArgs](https://github.com/JerichoJS/Membrane#communicationpackageargs).
- ### EventHandlingMechanism
  This class is instantiated globally under the variable name `eventHandler` within the project; see its section in [Utilities](https://github.com/JerichoJS/Membrane#eventhandlingmechanism) for more information
- ### AbstractMap
  Once again, see the [relevant section](https://github.com/JerichoJS/Membrane#abstractmap) of Utilities for a more complete treatment of the matter. The version presented here differs from the "utility form" only insofar as it also contains efficient `exportList` and `importList` methods, with a `this`-scoped boolean property `exportRefreshed` which identifies whether the current value of `export` accurately represents the map. Additionally, the higher-level `optionalExport` method will update the value of `export` through `exportList` if and only if `exportRefreshed` is false. This class is globally instantiated as `networkMap`, the central weighted ledger for peer associations; this is used in finding the most efficient peer routes across the network. Taking `Object.keys(networkMap.nodes)` or `Object.keys(networkMap.adjacencyList)` will render a list of all nodes found within the current network.
- ### PeerConnection
  The `peerConnection` class acts as a high-level wrapper for the RTCDataChannel and RTCPeerConnection APIs, facilitating abstract interaction between nodes, such as routing, "authenication,<!---->" and updating the networkMap. Every connection made within this module is represented as an instance of this wrapper.
  #### SDP Exchange
  - #### PeerConnection.prototype.makeOffer
     To be called on a newly-instantiated channel; acts as a synchronous `createOffer` ICE candidate aggregator, eventually providing an SDP package appropriate to the peer's `transport.connection` and ready for exchange.
  - #### PeerConnection.prototype.receiveOffer
    Accepts a package of SDP generated by another peer's `makeOffer`, simmilarly aggregating the candidates generated by RTCPeerConnection's `createAnswer`, returning the total answer.
  - #### PeerConnection.prototype.receiveAnswer
    Accepts the answer generated by `peerConnection.prototype.receiveOffer` and commits it to the peerConnection, completing the SDP exchange cycle and readying the connections for data exchange.
    ____
  - #### makeDefiniteRoute
    Accepts the hiddenAlias of an existing node within the network map and a level of authentication, known as `desiredPermissions`. Instantiates a new `peerConnection`, generates an offer through `makeOffer`, bundles this up into an appropriate package, and sends it via `detatchedRoute` to the nearest intermediary in the route to `destination`. The function then awaits one of three outcomes--`routeAccepted`, `routeRejected`, or `routeInaccessible`, and responds accordingly, either preparing the channel for data transfer or killing it and alerting the user.
  - #### comprehendProspectiveRoute
    Accepts a complete routing package, minus the `type` header, instantiates a new peer, attempts to add the SDP contained within the package, and, assuming this action has been successful, uses `CONFIG.communication.routeAcceptHeuristic`<!----> on the package to determine whether or not to persist the connection and formulate a response or to terminate the initialization protocol and destroy the peer.
  - #### handleMessage
    The central drain through which all packages recieved by the peer are aggregated; the code is quite exceptionally straightforward, but would be downright tedious, and certainly wasteful to display here, given its length. For a precise overview of the ways messages are handled, see <!--handleMessage code-->.
  - #### weaklyValidateMessage
    Ensures that all received messages conform to the standards outlined by `CONFIG.constants.packageArgs`<!---->, ultimately returning a boolean representing whether or not a message may be trusted in this respect.
  - #### messageMethods
    Holding a name whose sentiment has largely been eroded from its original purpose, this section holds merely two distinct message handling methods: the two requied for a proper symmetric peer handshake; namely, 
    - `invokerIntroduction` - Used on the package of the same name provided by the `voluntary` peer (that which initially "requested" the route,) this method applies several essential data proivded by the first peer, and eventually bundles up its own `reciprocalAlignment` package for this peer, containing a copy of the current networkMap, if the peer claims to be without one, and its own aliases.
    - `reciprocalAlignment` - Employed simmilarly to the former, but instead on packages of its own name; accepts and parses the networkMap and aliases provided by its peer, eventually reciprocally adding a live peer and completing the initialization exchange.
  - #### close
    Forcefully closes a peer and all its facilities, including enrolled registers and ledgers, and alerting the network of the death through the `gossipTransport`<!---->.
  - #### stabilizeLink
    Called in times of potential network instability, this function only ever takes effect if a given node has less than two remaining live peers. The peer will enter a loop of vigorously searching for a stable contact, preferably one as distant from it as possible, in order to reinforce the bonds holding the network together. The sequence will truly halt only when no feasible, unconnected peers remain, or else the peer successfully adds another connection.
- ### Floundering
  If a peer ever becomes fully disconnected from the network, it will begin the violent `flounder` procedure, entirely wiping its networkMap and performing a `serverHardRestart`, thus flailing around aimlessly until it is finally reentered into the network.
- ### GossipTransports
  Yet another critical mechanism within the network, that which keeps all (contiguous) parts synchronized and prevents total descent into chaos, is the `GossipTransport`. Gossip is the lifeblood of the network, constantly propagating and pulsing from node to node, alerting the entire entwork of every slight reconfiguration. As it is currently estabished, the `gossipTransport` (that is the global instantiation of `GossipTransport`) communicates exactly two species of change: networkMap weight (calculated off of the routing penalties found within `CONFIG.constants.violationWeightPenalties` and typically assigned through the `shiftNodeWeight` function), and topological reconfiguration, i.e. node or edge addition or deletion. All gossip flow is centrally regulated by the instance's `propagationPulse` interval. The function defined within here serves quite a critical role: determining exactly which items of gossip to commit every round. This is decided by looking at each defined `type`'s `iterModulo`; if `this.pulseIterations` % `iterModulo` equals zero, that type's name will be pushed to the `propagationStack`, wherefrom all uncommitted items of gossip will be aggregated into a single bundle and distributed throughout the network via `propogateAll` <!---->.
  - #### Types
    Every gossip transport has zero or more `types`, as stored within `this.types` and registered through the `addType` method. Types act as distinct, seggregated ledgers which store gossip intended for different purposes, to allow more fine-grained control over distribution of data. For instance, within the default Membrane script, two different types are registered to `gossipTransport`: `topology` and `weight`. The former is registered with only one parameter, defaulting it to dispatch every propagation interval, whereas the second is assigned a value of 100, making it propagate once every hundred runs, or every ten seconds. If a non-numeric, non-nullish value is provided for the modulo, a unique trigger function will be returned within the result of the function called `propagate`, and type will never automatically dispatch. Assuming a type is added successfully, `addType` will return a set of "trigger functions"--an object with properties `addGossip`, `remove`, and, optionally, `propagate`. AddGossip accepts precisely one argument, the piece of data to be propagated, which will be pushed to that type's buffer and eventually dispatched. Calling `remove` will immediately and irrevocably destroy the the type and all associated values, and `propagate` will immediately progagate the complete contents of the type with no regard for dispatch cycles.
  - #### Parsers
    Parsers act as perfect complements to types, and, as such, are absolutely useless unless at least one member of the network has registered and actively dispatches the corresponding type. Parsers are called over gossip both on immediately adding it through `addGossip` and on recieving it, through `consumeGossip`. If no parser is defined for a recieved type, a `default` parser is used, allowing the gossip to continue propagation but having absolutely no direct influence on the function of the recieving node. The `addParser` function is relatively extensible, allowing on-the-fly fine-grained registry; however, because of this, it is also unusually complex. The only strictly required argument is `type`, the type of gossip to respond to. However, a parser registered in this fashion will be rendered perfectly inert, not even performing the function of the `default`. The optional `useDefault` argument determines whether the parser shoudl first apply the `default` transform on the data before passing it to an optional `parserCallbacK`. The `parserCallback`, if defined, will recieve at least the entire `block` of `gossip`, as well as its `type` (typically redundant unless a parser is used adaptively for several types). If `useDefault` is set to true, the callback will recieve two more items, `committable` and `unknown`. The latter contains every component of the block containing data not already present within "`knownFacts`"; the former is merely a copy of the latter with all exteraneous data stripped out (having removed any values not specified by `constantArgs`). Regarding `constantArgs`, this list specifies which pieces of data are relevant to the actual fact, so that the parser can decide whether or not it already "knows" about a particular convolution, and therefore should not gossip about. If this argument is ommitted, the value defaults to the `Object.keys()` of the first member of the `block`. The final parameter, `preliminaryVerification` is only ever used if `useDefault` is set to true. In this situation, it acts as an individual discriminator function, to be run over every member of `block`. If a given item fails, it is withheld from the buffer and entirely forgotten.
  - #### `propagateAll`
    The `propagateAll` function propagates every item of a specific type of gossip simultaneously, bundling them all up into contiguous packages. If the total size of the block exceeds 16 KiB, it will be split into several, equally-sized packages in order to preserve transfer speed. After having distributed the complete block to each live peer, the buffer will be wiped completely clean.
- ### detatchedRoute
  This function attempts to `findNextHop` to the provided `destination` parameter. Assuming a route exists within the network between this initiator and the destination, a package will be sent to the computed neares intermediary containing the collective `rest` of the `content` `standardSend`<!----> arguments.
- ### makeServerLink
  The makeServerLink function 
- ### init
  - ####
- ### Status trackers
  This project employs two primary sets of connection status trackers--`livePeers` and `authPeers`. Each is modified through the appropriate `add` or `remove` method (i.e. `addAuthPeer` and `removeAuthPeer`), and may be watched for changes with a callback via an `onXUpdated` method (i.e `onAuthPeersUpdated`). Livepeers comprises a list of all peers with which the client holds direct connections, with new peers automatically appended to it following successful initialization exchanges, and peers removed on explicit termination or death of their underlying dataChannels. AuthPeers acts as a softer abstraction on this, containing the hiddenAliases, rather than actual references, of all peers which have `advanced` <!---->send permissions.
- ### Authentication
  Authentication is essential to the underlying mechanism of this project. Within the scope of peerConnections, the term authentication is taken in a slightly different sense from the standard cybersecurity definition. Here, authentication means that a peer will readily accept `consumable` data--that which is presented to the user, rather than used internally.  
  Within membrane networks, peers do not connect solely for the purpose of explicit data exchange; the network constantly stabilizes<!----> by creating redundant routes between nodes in order to improve fault tolerance. Therefore, authentication is a necessary formalism to show that, not only are two nodes implicitly connected, but, too, they both agree to exchange consumable data. Practically, there are three main ways to establish an authenticated route: `peerConnection.prototype.makeDefiniteRoute` with "permissions" set to "advanced," `(instance of peerConnection).requestPermissionEscalation`, with the sole parameter "advanced," or the more dynamic hybrid of the two--`peerConnection.prototype.negotiateAgnosticConnection`, which will perform an escalation if a route already exists to the destination, and, if not, employ the makeDefiniteRoute method.
## Installation and Integation
The following sequence of commands will install and initiate the demo on any reasonably recent *nix-style operating system.
```shell
curl -LJo Membrane-v1.0.0.tar.gz https://github.com/Elijah-Bodden/Membrane/tarball/v1.0.0
tar xfv Membrane-v1.0.0.tar.gz --transform 's!^[^/]\+\($\|/\)!Membrane-v1.0.0\1!'
cd Membrane-v1.0.0/src/source/frontend
npm install
cd ../server
npm install
npm run deploy
```
To kill the pm2 daemon generated by `npm run deploy`, simply run `npm run kill`.
<!-- the following functions, etc, must be supplied -->
### What else you'll need
<!-- A functional server with appropriate endpoints + exchange methods (see /src/server -> wget -> cd -> npm install) -->
## Application Infrastructure

## Utilities
Within the root directory of this repository, you will find a subdirectory named simply `"Utilities."` Here I have compiled two of the most helpful independent utilities I wrote for this project. Each is remarkably small (just over 200 lines of code combined,) but they have proven absolutely invaluable to me in the later stages of the project; I hope they may do for someone else in my sharing. Both of these are stripped down versions of classes found in the `lib` source, and, as such, if you wish for full functionality at the expense of a few more resources, you can find classes of the same names within `index.js`.
- ### EventHandlingMechanism
    This lightweight utility acts as a fully functional event-dispatch and -signaling device, with both asyncronous, promise-based channels (invoked by the method `acquireExpectedDispatch`), and more traditional, callback-based modes (bound by calling `onReceipt`). After attatching listeners to a particular "`signalIdentifier`" by either method, you can then trigger their products by calling `dispatch` with the desired identifier as the first parameter. Into this function you may also optionally pass a second argument--an `externalDetail`--which will then be passed to every callback and returned with the value of every promise.
- ### AbstractMap
    This utility is perhaps slightly more straightforward than the former, acting simply as a lightweight, adjacency-list-based representation vector for undirected force maps (optionally weighted). Alongside standard methods for node and edge manipulation and weighting, I also include an efficient representation of Dijkstra's pathfinding algorithm, invoked by the function `precomputeRoutes`. The product of this algorithm is saved to the locally-scoped variable `distances`, and may be extracted further through the method `findNextHop`, which identifies the first intermediary between two desired nodes.
## Thanks
