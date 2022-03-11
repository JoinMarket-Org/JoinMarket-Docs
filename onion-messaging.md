## MESSAGE FORMAT USED on the ONION MESSAGING CHANNEL

( `||` means concatenation for strings, here)

These message channels are based on communication between Joinmarket nodes, where two classes of node, namely *directory nodes* and *maker* nodes, are accessible via Tor v3 onion addresses, and the remaining nodes connect to them (using a SOCKS5 proxy).

Messages are defined as in `jmdaemon.onionmc.OnionCustomMessage`. They are an encoded json struct, which always contains two fields: `line` and `type`. `line` is always a `str` and `type` is always an `int`. This `type` is the integer specified in this file as `*_MESSAGE_TYPE`s.

The messages (which, as per above, are serialized, encoded json) passed between connected nodes are sent as lines (i.e. delimited with newlines), using the twisted `LineReceiver` protocol class (see [here](https://twistedmatrix.com/documents/current/api/twisted.protocols.basic.LineReceiver.html)). The `LineReceiver` protocol was chosen to fit with the existing method used in the IRC messaging already (as well as its simplicity).

### MESSAGE TYPES:

```python
LOCAL_CONTROL_MESSAGE_TYPES = {"connect": 785, "disconnect": 787, "connect-in": 797}
CONTROL_MESSAGE_TYPES = {"peerlist": 789, "getpeerlist": 791,
                         "handshake": 793, "dn-handshake": 795}
JM_MESSAGE_TYPES = {"privmsg": 685, "pubmsg": 687}
```

### FORMAT OF MESSAGES:

The field `line` as described above, are text strings of this format:

```
from-nick || COMMAND_PREFIX || to-nick || COMMAND_PREFIX || cmd || " " || innermessage
```

(`COMMAND_PREFIX` is defined in `jmdaemon/protocol.py`)

Here, `innermessage` may be a list of messages (e.g. the multiple offer case) separated by `COMMAND_PREFIX`.

Note that this syntax (the part after `to-nick` in the above) will still be as was described in this repo, in [Joinmarket-messaging-protocol.md](https://github.com/JoinMarket-Org/JoinMarket-Docs/blob/master/Joinmarket-messaging-protocol.md#joinmarket-messaging-protocol).

Note also that there is currently no chunking requirement applied here.

### CONTROL MESSAGES

#### HANDSHAKE CONTROL MESSAGES

The message `handshake` is sent by any peer/node not configured to act as
directory node, to any other node it connects to, as the first message.
The message `dn-handshake` is sent by any peer/node which is configured to
act as a directory node, to any other node, as a response to the initial
`handshake` message.
(Notice that this configuration implies that directory nodes do not currently
talk to each other).

The syntax of `handshake` is:

```json
  {"app-name": "joinmarket",
   "directory": false,
   "location-string": "host:port",
   "proto-ver": 5,
   "features": {},
   "nick": "J5***"
  }
```

Note that `proto-ver` is the version specified as `JM_VERSION` in `jmdaemon.protocol`.
(It has not changed for many years, it only specifies the syntax of the messages).
The `features` field is currently empty but is provided for forwards compatibility, if directory nodes offer additional features.

The syntax of `dn-handshake` is:

```json
 {"app-name": "joinmarket",
  "directory": true,
  "proto-ver-min": 5,
  "proto-ver-max": 5,
  "features": {},
  "accepted": true,
  "nick": "J5**",
  "motd": "Information about directory node"
 }
```

 Non-directory nodes should send `handshake` to directory nodes, upon successfully connecting, and directory nodes should return the `dn-handshake` method with `true`
 for accepted, if and only if:
 * the protocol version is in the accepted range
 * the `directory` field of the peer is false
 * the `app-name` is the string `"joinmarket"`
 * the set of features requested is both recognized and accepted (currently: none)

 In case those conditions are met, return `"accepted": true`, else return
 `"accepted": false` and immediately disconnect the new peer.
 (in this rejection case, the remaining fields of the `dn-handshake` message do
 not matter, but can be kept as before for convenience).

In case of a direct connection between peers (neither are directory nodes),
the party which connects then sends the first `handshake` message, and the
connected-to party responds with their own `handshake`.

In this case, the connection should be accepted and maintained by the receiver
if and only if:
* the protocol version is identical
* the `directory` field of the peer is false
* the `app-name` is `"joinmarket"`
* the set of features is both recognized and accepted (currently: none)

otherwise the peer should be immediately disconnected.

ALL OTHER MESSAGES (control or otherwise, as detailed below), cannot be sent/
will be ignored until the above two-way handshake is complete.

#### OTHER CONTROL MESSAGES

The syntax of `peerlist` is:

```
nick || NICK_PEERLOCATOR_SEPARATOR || peer-location || "," ... (repeated)
```

i.e. a serialized list of two-element tuples, each of which is a Joinmarket nick
followed by a peer location.

`peerlist` may be sent by directory nodes to non-directory nodes at any time,
but currently it is sent according to a specific rule described below.

#### LOCAL CONTROL MESSAGES

There are three messages created inside the joinmarket messaging daemon, in response to network level events, namely the `connect`, `connect-in`, and `disconnect` events. These are used to update the *state* of new peers, or existing peers that we have recorded as connected at some point, to ourselves.


### PARSING OF RECEIVED JM_MESSAGES

Messages are received as lines via the `OnionLineProtocol` instance for the given connection.
The line is first deserialized into a json object, containing the fields `line` and `type` as outlined above.
The origin of the message is defined as either inbound or outbound. If the message is a control message, the connection information (which can be host:port or the defined reachable onion location, or the string "00" for ourself) may be used to update the state of the peer. If it is a Joinmarket message then the `nick` in the line (as outlined above) is used to determine where to send the message (i.e. directory nodes broadcast to all connected non-directory peers for `PUBMSG` and forward to a specific peer for `PRIVMSG` but see below for the special behaviour in regards to `PRIVMSG` on these message channels).

The resulting messages of the Joinmarket type are passed into normal message channel processing, and should be
identical to those coming from IRC.


#### GETTING INFORMATION ABOUT PEERS FOR DIRECT CONNECTIONS

To avoid passing huge lists of peers around, the directory node takes a "lazy" approach to
sharing connection info between peers:

When Peer J51 (maker) asks to privmsg J52 (taker) (which it discovered when receiving a privmsg from J52, usually
here that would be in response to a `!orderbook` pubmsg by J51), the directory node does as instructed,
but then sends also a `peerlist` message to J51, containing the full network location of J52, *if it is available*.

#### Conditions under which network location is and is not available.

In the handshake, peers who do *not* serve onions, will use `"NOT-SERVING-ONION"` as their `location-string` in the handshake. This lets the party receiving the connection know that there is no available network location. In this case, the directory knows that it cannot send connection information for that peer, to other peers. So in the case above, J51 will not receive a `peerlist` message containing connection information for J52 (since it does not exist). However, when J52 sends a privmsg to J51 (usually this means responding with `!fill`), the situation is opposite: J52 *does* have a non-fake network location (*.onion) recorded from its handshake with the directory node, so the directory node will indeed forward that *.onion* address in a `peerlist` message to J52.

Given this new information, J52 opportunistically tries to connect to J51 directly at its .onion, and if the network connection succeeds, sends a handshake to J51. If J51 responds with acceptance, the direct messaging connection is established, and from then on, until J52 and J51 see a disconnect event for that network peer, they will divert any `privmsg` to that party to use the direct connection (either inbound or outbound) instead of the directory node.


