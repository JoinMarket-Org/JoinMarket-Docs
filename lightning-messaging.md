## MESSAGE FORMAT USED on the LN-ONION CHANNELS

( `||` means concatenation for strings, here)

These message channels are based on communication between [c-lightning](https://github.com/ElementsProject/c-lightning) nodes,
accessible over Tor v3 onion addresses.

Messages conveyed between directly connected nodes with `sendcustommsg`
have the format described [here](https://lightning.readthedocs.io/PLUGINS.html#custommsg).

Note in particular, that `type` is a two byte hex-encoded string
that prepends the actual message (also hex-encoded), without any intervening
length field. This `type` is the integer specified in this file as `*_MESSAGE_TYPE`s.

### MESSAGE TYPES:

```
LOCAL_CONTROL_MESSAGE_TYPES = {"connect": 785, "disconnect": 787, "connect-in": 797}
CONTROL_MESSAGE_TYPES = {"peerlist": 789, "getpeerlist": 791,
                         "handshake": 793, "dn-handshake": 795}
JM_MESSAGE_TYPES = {"privmsg": 685, "pubmsg": 687}
```

Note also that the integer types *must* be odd for custom messages (the "it's OK to be odd" principle).

### FORMAT OF MESSAGES:

In text, the messages we send via this mechanism are of this format:

```
from-nick || COMMAND_PREFIX || to-nick || COMMAND_PREFIX || cmd || " " || innermessage
```

(`COMMAND_PREFIX` is defined in `jmdaemon/protocol.py`)

Here, `innermessage` may be a list of messages (e.g. the multiple offer case) separated by `COMMAND_PREFIX`.

Note that this syntax will still be as was described in this repo, in [Joinmarket-messaging-protocol.md](https://github.com/JoinMarket-Org/JoinMarket-Docs/blob/master/Joinmarket-messaging-protocol.md#joinmarket-messaging-protocol).

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
json serialized:
  {"app-name": "joinmarket",
   "directory": false,
   "location-string": "hex-key@host:port",
   "proto-ver": 5,
   "features": {},
   "nick": "J5***"
  }
Note that `proto-ver` is the version specified as `JM_VERSION` in jmdaemon.protocol.
(It has not changed for many years, it only specifies the syntax of the messages).
The `features` field is currently empty.

The syntax of `dn-handshake` is:
json serialized:
 {"app-name": "joinmarket",
  "directory": true,
  "proto-ver-min": 5,
  "proto-ver-max": 5,
  "features": {}
  "accepted": true,
  "nick": "J5**",
  "motd": "Information about directory node"
 }

 Non-directory nodes should send `handshake` to directory nodes,
 and directory nodes should return the `dn-handshake` method with `true`
 for accepted, if and only if:
 * the protocol version is in the accepted range
 * the `directory` field of the peer is false
 * the `app-name` is joinmarket
 * the set of features requested is both recognized and accepted (currently: none)
 * the nick used by this entity/bot across all message channels, used for cross-channel message spoofing protection

Notice that more than one nick is NOT allowed per LN node.

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
* the `app-name` is joinmarket
* the set of features is both recognized and accepted (currently: none)

otherwise the peer should be immediately disconnected.

ALL OTHER MESSAGES (control or otherwise, as detailed below), cannot be sent/
will be ignored until the above two-way handshake is complete.

#### OTHER CONTROL MESSAGES

The syntax of `peerlist` is:

nick || NICK_PEERLOCATOR_SEPARATOR || peer-location || "," ... (repeated)

i.e. a serialized list of two-element tuples, each of which is a Joinmarket nick
followed by a peer location.

`peerlist` may be sent by directory nodes to non-directory nodes at any time,
but currently it is sent according to a specific rule described below.

#### LOCAL CONTROL MESSAGES

There are two messages passed inside the plugin, to to the joinmarket messaging daemon,
in response to events in lightningd, namely the `connect` and `disconnect` events
triggered at the LN level. These are used to update the *state* of existing peers
that we have recorded as connected at some point, to ourselves.

The mechanisms here are loosely synchronizing a database of JM peers, with obviously the directory
node(s) acting as the data provider. It's notable that there are no guarantees of accuracy or
synchrony here.

### PARSING OF RECEIVED JM_MESSAGES

The text will be utf-encoded before being hexlified, and therefore the following actions are needed of the receiver:

Extract the peerid of the sender, and the actual message, received as encoded json:

* peerid: json.loads(msg.decode("utf-8"))["peer_id"]
* message: json.loads(msg.decode("utf-8"))["payload"]
(which is prepended by a two byte message_type; see below).

##### peerid:

This field comes in three potential forms (as hex):
"00" : special null peerid indicating a local control message (see above).
"hex-key": peerid without connection information, allowing us to record the existence of a peer,
           but not to send messages to it directly (only via the directory node).
"hex-key@host:port": peerid with connection information, allowing us to attempt to connect to it,
           and send private messages to it directly.

##### payload:

This is parsed by:

1. Take the first two hex-encoded bytes and convert to an integer: this is the
                   message type. Take the remaining part of the hex string and unhexlify,
                   converting to binary, then .decode("utf-8") again, converting to a string.
                   (This means encoding is sometimes very inefficient, if the underlying string actually
                   contains hex or other encoding, but can't really be changed until the underlying
                   Joinmarket messaging protocol is changed.)

2. Split the decoded string by `COMMAND_PREFIX` and parse as `from nick, to nick, command, message(s)`
           (see above for syntax).

The resulting messages can be passed into Joinmarket's normal message channel processing, and should be
identical to that coming from IRC.

However, before doing so, we need to identify "public messages" versus private, which does not
have as natural a meaning here as it does on IRC; we impose it by using a to-nick value of `PUBLIC`
and by sending the message_type `687` to the Lightning RPC instead of the default
message_type `685` for privmsgs to a single counterparty. This will instruct the directory node
to send the message to every peer it knows about.

#### GETTING INFORMATION ABOUT PEERS FOR DIRECT CONNECTIONS

To avoid passing huge lists of peers around, the directory node takes a "lazy" approach to
sharing connection info between peers:

When Peer J51 asks to privmsg J52 (which it discovered when receiving a privmsg from J52, usually
here that would be in response to a `!orderbook` pubmsg by J51), the directory node does as instructed,
but then sends also a `peerlist` message to J51, containing the full network location of J52.

Given this new information, J51 opportunistically tries to connect to J52 directly and if the network
connection succeeds, sends a handshake to J52. If J52 responds with acceptance, the direct messaging
connection is established, and from then on, until J51 sees a disconnect event for that network peer,
he will divert any `privmsg` to that party to use the direct connection instead of the directory node.


