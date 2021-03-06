Switch Core Channels
====================

A telehash switch must implement the channels defined here in order to fully support the protocol.  These channels are the essential connectivity foundation, including the [DHT](dht.md) basics and NAT hole punching techniques.

To understand how these channels are used and interact with each other, jump to the [Connection Flows](#connecting) for examples of all of the ways one hashname connects to another.

## Core Channels

The following values for `type` are for unreliable channels that are used by switches to provide and maintain connectivity between instances. They are part of the core spec, and must be implemented internally by all switches:

  * [`seek`](#seek) - given a hashname, return any pointers to other hashnames closer to it (DHT)
  * [`link`](#link) - request/enable another hashname to return the other in a `seek` request (DHT)
  * [`peer`](#peer) - ask the recipient to make an introduction to one of its peers
  * [`connect`](#connect) - a request asking to try to open a connection to a given hashname (result of a `peer`)
  * [`bridge`](#bridge) - how a switch can proxy traffic for two hashnames to enable full connectivity
  * [`path`](#path) - how two switches prioritize and monitor network path information


<a name="seek" />
### `"type":"seek"` - Finding Hashnames (DHT)

The core of Telehash is a basic [Kademlia-based DHT](dht.md), the bulk of the logic is in the rules around maintaining a mesh of lines and calculating distance explained there. When one hashname wants to connect to another, it recursively sends `seek` requests to find closer and closer peers until it's discovered or there are none closer. The seek request contains a `"seek":"hex-value"` that is always a prefix of the hashname that it is trying to connect to.

When initating a new connection, the first seek requests should always be sent to the closest hashnames with active [links](#link).  Then the switch recursively sends seeks to the closest hashnames to the target until it discovers it or cannot find any closer.  It is suggested that this recursive seeking process should have at least three threads running in parallel to optmize for non-responsive nodes and round-trip time.  If no closer nodes are being discovered, the connection process should fail after the 9 closest nodes have been queried or timed-out.

Only the prefix hex value is sent in each seek request to reduce the amount of information being shared about who's seeking who. The value then is only the bytes of the hashname being saught that match the distance to the recipient plus one more byte in order for the recipient to determine closer hashnames.  So if a seek is being sent to  "1700b2d3081151021b4338294c9cec4bf84a2c8bdf651ebaa976df8cff18075c" for the hashname "171042800434dd49c45299c6c3fc69ab427ec49862739b6449e1fcd77b27d3a6" the value would be `"seek":"1710"`.

The response is a compact `"see":[...]` array of addresses that are closest to the hash value (based on the [DHT](dht.md) rules).  The addresses are a compound comma-delimited string containing the "hash,cs,ip,port" (these are intentionally not JSON as the verbosity is not helpful here), for example "1700b2d3081151021b4338294c9cec4bf84a2c8bdf651ebaa976df8cff18075c,1a,123.45.67.89,10111". The "cs" is the [Cipher Set](cipher_sets.md) ID and is required. The ip and port parts are optional and only act as hints for NAT hole punching.

Only hashnames with an active `link` may be returned in the `see` response, and it must always include an `"end":true`.  Only other seeds will be returned unless the seek hashname matches exactly, then it will also be included in the response even if it isn't seeding.

<a name="link" />
### `"type":"link"` - Enabling Discovery (DHT)

In order for any hashname to be returned in a `seek` it must have a link channel open.  This channel is the only mechanism enabling one hashname to store another in its list of [buckets](dht.md) for the DHT.  It is bi-directional, such that any hashname can request to add another to its buckets but both sides must agree/maintain that relationship.

The initial request:

```json
{
  "c":1,
  "type":"link",
  "seed":true,
  "see":["c6db0918a767f00b9841f4366ade7ffc13c86541c40bf0a1612e939988fdefb0,1a,184.96.145.75,59474"]
}
```

Initial response, accepting the link:

```json
{
  "c":1,
  "seed":false,
  "see":["9e5ecd193b14abaef376067f80f442be97f6f3110abb865398c2a6ec83a4ee9b,2a"]
}
```

The `see` value is the same format as the `seek` response and pro-actively sends other seeds to help both sides establish a full mesh.  The see addresses should all be closer to the recipient, but if there are none then further addresses may be sent to help bootstrap.  The `seed` value indicates wether the sender/recipient wants to act as a seed and be included in `seek` requests, otherwise it will only be included in the see response when it matches the seek exactly.

In the initial response or at any point an `end` or `err` can be sent to cancel the link, at which point both sides must remove the corresponding ones from their DHT.

The link channel requires a keepalive at least once a minute in both directions, and after two minutes of no incoming activity it is considered errored and cancelled.  When one side sends the keepalive, the other should immediately respond with one to keep the link alive as often only one side is maintaining the link.  Links initiated without seeding must be maintained by the requestor.

The keepalive requires only the single key/value of `"seed":true` or `"seed":false` to be included to indicate its seeding status. This keepalive timing is primarily due to the prevalance of NATs with 60 second activity timeouts, but it also serves to keep only responsive hashnames returned for the DHT.

Details describing the distance logic, maintenance, and limits can be found in [DHT](dht.md) reference.

#### Advertising Bridge Capacity

When either side of the link is willing to [bridge](#bridge) packets for the other, it must include a `"bridges":["ipv4","ipv6"]` of the network types that it supports bridging for.  This acts as an idicator to the recipient that it can make bridge requests for that network path type when needed. Bridges can be advertised or updated at any time, an empty array cancels any bridge advertisements.

```json
{
  "c":1,
  "seed":true,
  "bridge":["ipv4","ipv6","http"]
}
```


<a name="peer" />
### `"type":"peer"` - Introductions to new hashnames

For any hashname to send an open to another it must first have one of its public keys, so all new opens must be "introduced" via an existing line. This introduction is a two step process starting with a peer request to an intermediary. Since new hashnames are discovered only from another one in the `see` values, the one returning the see is tracked as a "via" so that they can be sent a `peer` request when a connection is being made to a hashname they sent. This also serves as a workaround if any NAT exists, so that the two hashnames can send a packet to each other to make sure the path between them is open, this is called "hole punching." 

A peer request requires a `"peer":"851042800434dd49c45299c6c3fc69ab427ec49862739b6449e1fcd77b27d3a6"` where the value is the hashname the sender is trying to reach. The BODY of the peer request must contain the binary public key of the sender, whichever key is the highest matching [Cipher Set](cipher_sets.md) as signalled in the original `see`.  The recipient of the peer request must then send a connect (below) to the target hashname (that it already must have an open line to).

The peer channel that is created remains active and serves as a path for tunneled packes to/from the requested hashname, those tunneled packets will always be attached as the raw BODY on any subsequent sent/received peer channel packets.

If a sender has multiple known public network paths back to it, it should include an [paths](#paths) array with those paths, such as when it has a valid public ipv6 address.  Any internal paths (local area network addresses) must not be included in a peer request, only known public address information can be sent here.  Internal paths must only be sent in a [path](#path) request since that is private over a line and not exposed to any third party (like the peer/connect flow is).

```json
{
  "c":10,
  "type":"peer",
  "peer":"ed1a50bdd08846ee9ed504ba59469a843b234dc9e6e56470b76ff8839b08039c",
  "paths":[{"type":"ipv4","ip":"12.14.16.18","port":24242}]
}
BODY: ...sender's binary public key...
```

<a name="connect" />
### `"type":"connect"` - Connect to a hashname

The connect request is an immediate result of a `peer` request and must always attach/forward the same original BODY it as well as a [paths](#paths) array identifying possible network paths to it.  It must also attach a `"from":{...}` that is the [Cipher Set](cipher_sets.md) keys of the peer sender, identical format as to what is sent as part of an `open`.

The recipient can use the given public key to send an open request to the target via the possible paths.  If a NAT is suspected to exist, the target should have already sent a packet to ensure their side has a path mapped through the NAT and the open should then make it through.

When generating a connect, the switch must always verify the sending path is included in the paths array, and if not insert it in if it's a public path. This ensures that the recipient has at least one valid path and speeds up path discovery since no additional [`path`](#path) round trip such as for an IPv6 one.

#### Connect Handling

The recipient of a connect is being asked to establish a line with the included hashname by a third party, and must be wary of the validity of such requests, both checking the included BODY against the `from` info to verify the hashname and matching CSID, as well as tracking the frequency of these requests to reduce outgoing unsolicited requests. There must be no more than one open packet sent per destination host per second.

The generated `open` should always be attached as a BODY and sent back in response on the new connect channel as well, which relays it back to the original `peer` request to guarantee connectivity.  Subsequent packets on the connect channel work identically to the peer one, acting as a tunnel with the BODY being the raw tunneled packet.

The switch acting as the relay between a `peer` and `connect` must limit the rate of tunneled packets to no more than 5 per second in either direction, and never have more than one peer-connect pair active between two hashnames.  This enables the two hashnames to privately negotiate other connectivity, but not use it's bandwidth as an open bridge.

#### Auto-Bridge

If this switch is willing to act as a bridge, as soon as it has detected a tunneled [line](network.md#line) in both directions it should internally set up a [bridge](#bridge) and always include a `"bridge":true` on every tunneled packet thereafter.  Either side of the tunnel when seeing this flag should then treat the channel's sending path as that of the tunneled packet, and subsequent line packets to that destination will be bridged to the other source.

<a name="bridge" />
### `"type":"bridge"` - Shared Bandwidth

The bridge channel is used to enable other hashnames (either anyone, or just specific trusted ones) to proxy the traffic for a single line through the hosting switch when the two parties of the line cannot communicate directly (NATs, firewalls, different network types, etc).  The supporting switch will receive the line packets and immediately send them all to a different destination instead of processing them.

A `bridge` request looks like:

```json
{
	"c":1,
	"type":"bridge",
  "to":"be22ad779a631f63336fe051d5aa2ab2",
  "path":{"type":"ipv4", "ip":"1.2.3.4", "port":5678},
  "from":"69ab427ec49862739b6449e1fcd77b27"
}
```

The `to` value is the incoming line id, when any packet is received by the bridge switch with that id the packet is sent to the specified [path](network.md#paths).  When the request is confirmed the channel will be ended without an error, otherwise an `"err":"reason"` will be returned.

When any line id coming into the switch matches the `from` value it's resent to the network path that the bridge was created from.  Bridges should be persisted until the hashname that created it goes offline.

This enables a supporting switch to do essentially no work in bridging packets as it can process them outside any encryption.  To prevent circular loops, all bridged packets must have a hash calculated and temporary cache of the values stored to detect any repeat packets that should be dropped.


<a name="path" />
### `"type":"path"` - Network Path Information

Any switch may have multiple network interfaces, such as on a mobile device both cellular and wifi may be available simutaneously or be transitioning between them, and for a local network there may be a public IP via the gateway/NAT and an internal LAN IP. A switch should always try to discover and be aware of all of the networks it has available to send on (ipv4 and ipv6 for instance), as well as what network paths exist between it and any other hashname. 
 
An unreliable channel of `"type":"path"` is the mechanism used to discover, share, prioritize and test any/all network paths available.  For each known/discovered network path to another hashname a `path` is sent over that network interface including an optional `"priority":1` (any positive integer, defaults to 0) representing its preference for that network to be the default.  The responding hashname upon receiving any `path` must return a packet with `"end":true` and include an optional `"priority":1` with its own priority for that network interface is to receive packets via.

Every path request/response must also contain a `"path":{...}` where the value is the specific path information this request is being sent to. The sending switch may also time the response speed and use that to break a tie when there are multiple paths with the same priority.

A switch should send path requests to its seeds whenever it comes online or detects that its local network information has changed in order to discover its current public IP/Port from the response `path` value in case it's behind a NAT.

Additional path requests only need to be sent when a switch detects more than one (potential) network path between itself and another hashname, such as when its public IP changes (moving from wifi to cellular), when line packets come in from different IP/Ports, when it has two network interfaces itself, etc.  The sending and return of priority information will help reset which paths are then used by default.

A [paths](#paths) array should be sent with every new path channel containing all of the sender's paths that they would like the recipient to use. Upon receiving a path request containing an `paths`, the array should be processed to look for new/unknown possible paths and those should individually be sent a new path request in return to validate and send any priority information.

#### Path Detection / Handling

There are two states of network paths, `possible` and `established`.  A possible path is one that is suggested from an incoming `connect` or one that is listed in an `paths` array, as the switch only knows the network information from another source than that network interface itself.  Possible paths should only be used once on request and not trusted as a valid destination for a hashname beyond that.

An established path is one that comes from the network interface, the actual encoded details of the sender information.  When any `open` or `line` is received from any network, the sender's path is considerd established and should be stored by the switch as such so that it can be used as a validated destination for any outgoing packets.  When a switch detects that a path may not be working, it may also redundantly send the hashname packets on any other established path.


<a name="connecting" />
## Connection Flows

> NOTE: this section is being added and a work in progress, each section needs a sequence chart

<img src="peers.png" width="500" />

### Direct (no DHT)

`A` shares [seed info](seeds.md) (and is not behind a NAT), `B` uses seed info to send an open directly.

### Meshing

To be reachable, every hashname must minimally mesh.  Uses [link](#link).

### Seek

`A` is seeking `B` via `C`.

* seek `A->C` with `B` info returned
* peer `A->C` for `B`, and empty packet `A->B` for possible NAT
* connect `C->B` with `A` info
* open `B->A` directly and `A->B` in response

### Seek (failed direct)

When a UDP direct connection is not possible or fails, exchange paths to look for alternative ones.

* seek `A->C` with `B` info returned
* peer `A->C` for `B`, and empty packet `A->B` for possible NAT
* connect `C->B` with `A` info
* open `B->C` back over the connect
* relay open `C->A` back over the peer, and `A->C->B` in response
* path over the `C` relay to look for direct network info

### Seek (bridge via relay)

When no paths are available, `C` can offer to directly bridge all line packets.

* seek `A->C` with `B` info returned
* peer `A->C` for `B`, and empty packet `A->B` for possible NAT
* connect `C->B` with `A` info
* open `B->C` back over the connect
* relay open `C->A` back over the peer, and `A->C->B` in response
* the `C` signals to `"bridge":true` on the relay'd packets
* `A` and `B` use the network path for `C` as the one for each other directly

### Seek (general bridge)

When no paths are available, elect a [bridge](#bridge).

* seek `A->C` with `B` info returned
* peer `A->C` for `B`, and empty packet `A->B` for possible NAT
* connect `C->B` with `A` info
* open `B->C` back over the connect
* relay open `C->A` back over the peer, and `A->C->B` in response
* path over the `C` relay to look for direct network info
* `A` or `B` fail to find a supported or working network path, create a bridge

