Architectural Overview of Joinmarket as of Early 2019
===

This is a rough overview for people with a good technical background, who are curious to know the details of Joinmarket's codebase and algorithms, but who don't want to directly wade into the code — yet. It is also specific to the time mentioned above, things are of course changing — always.

The code is in Python: currently version 2.7 and 3.5+ compatible.
The most significant Python dependency is the "Twisted" package - this is used to make the code asynchronous only, i.e. there are, now, no threads used anywhere (except in an orderbookwatcher add-on script which is just used to view the market, separate from the opperation of Joinmarket itself).

Requirements: Bitcoin Core, runs on most popular Linuces, MacOS (afaik, maybe . . . possibly), but not on Windows unless by way of an instance of a/the Linux subsystem.

**joinmarket-clientserver** : The reason for this slightly strange name is to isolate two very different functionalities: the "business logic" of the agents (makers and the taker), and their reference to and communication with the wallet, from the end-to-end ("e2e") encrypted message passing (and some plaintext, too) with the other agents/peers over multiple redundant "message channels" (currently IRC only, but multiple); this messaging part ("server") knows the "joinmarket protocol", i.e.the language in which agents talk to each other, but knows nothing about bitcoin. So crudely the architecture is:

```
Bitcoin Core <--rpc--> Joinmarket wallet 
                          ^
                          |
                         code
                          |
                          v
          "client"(business logic of taker or maker)
                          ^
                          |
             (same *or* different process/machine)
                          |
                          v
        twisted AMP calls-> "server" (handles messaging+JM protocol)
                          ^
                          |
          (network, usually including Tor)
                          ^
                          |
                          v
                     IRC / other messaging server
```                          

4 Python packages:
====
1. **jmbase**: . . . currently contains very little (although it would be an improvement to contain more), specifically the twisted AMP communication protocol commands. We use a 2-way asynchronous communication protocol between the "client" and the "server".
2. **jmbitcoin**: As you'd expect; handles BIP32, signing and verifying, and tx construction. It uses a Python binding to libsecp256k1 ('coincurve').
3. **jmclient**: This implements the client protocol side of the AMP 2-way communication, and implements the business logic of the maker and taker agents, but also implements the underlying bitcoin wallet (including communication with Bitcoin Core over rpc) and the agents refer to this. So jmclient therefore has a dependency on jmbitcoin.
4. **jmdaemon**: . . . implements daemon (server) side of the AMP 2-way communication, implements the e2e encryption of messages, contains abstract message channel classes, contains IRC as the only current implementation of this, includes networking code for tor, and includes basic specifications for the "JM protocol" (i.e. the language in which the agents talk to each other. Note: jmdaemon does not have a dependency on jmbitcoin: it uses a Python binding to libsodium ('libnacl').

The actual tools for the user are outside these packages, in the `/scripts` subdirectory.


Note About Separation of jmdaemon and jmclient:
===============================================
It's possible to run the daemon part as a separate process (`scripts/joinmarketd.py`); this isolates the network access from the bitcoin signing code. At least one user/enthusiast is actually running JM like this today, but it isn't advertised as a feature because a little more work is needed to make it easy to be secure. SSL between the two can be done with selfsigned certificates but the code is only elementary and has not been worked on; it would probably be quite easy to fix this up into a more usable form. Obviously this is most interesting if done on different machines, not just different processes.

(To be clear, the default way of running has both the daemon and the client in the same process.)


How Bots Communicate over the Message Channels
==============================================
First note that the detailed communication protocol is documented at [joinmarket-messaging-protocol](https://github.com/JoinMarket-Org/JoinMarket-Docs/blob/master/Joinmarket-messaging-protocol.md), hereafter "JMP doc" for brevity.
On startup the client side code will generate an ephemeral secp256k1 private key and use it to generate a pubkey and then a base58 encoded short "nick" (as in "nickname", prepended with a 'J' and a version number) and send this as part of its startup parameters to the daemon (communication) side. The purpose of this is to allow, authentication / prevent nick spoofing, which is of special importance because we use multiple redundant message channels and, otherwise, an attacker could spoof a user's nick successfully on a different message channel than the one he's actually active on. To prevent this, each message, whether plaintext or encrypted must be accompanied by a signature against the public key corresponding to the base58 nick. See the JMP doc referenced above for details.

Making the above work does have a downside in that each message created and read requires a callback from the daemon to the client to verify or sign each individual message that is being exchanged.

Once communication between two bots starts (again see JMP doc) for a coinjoin, they do a DH key exchange using ephemeral libsodium keys, but this needs MITM prevention; this is achieved by having a signature of the DH key created using the private key corresponding to one of the output addresses in the transaction.
Once this authentication is completed, the rest of the inter-bot communication is completed using E2E encrypted (and base64 encoded for transmission over IRC, somewhat suboptimal!) messages (and still using the per-message signing to avoid spoofing, as described above).

Messages received over the messaging channel are heavily checked for conformance to the messaging protocol, inside the `jdaemon` code (both in `jmdaemon/irc.py` and `jmdaemon/message_channel.py`) to avoid unexpected behaviour, in particular crashing. This is of course particularly important to avoid for the long-running Maker bots.

Note specific to IRC: since public IRC servers have, naturally, some fairly significant measures against flooding, the IRC code throttles messages to a fairly low throughput, something like 400 bytes per second. This makes it less than practical to attempt very large numbers of counterparties.

Note on Using Tor
=================
A high degree of anonymity is clearly highly desirable for communicating with external message servers. (Note however it's not an absolute requirement for delinking as it is in things like chaumian server coinjoin or tumblebit, because the server is not constructing the transaction and most information passed is e2e encrypted). This is facilitated in our IRC code. Our other "external" connection is to Bitcoin Core RPC. This can be done over Tor also if on a different machine, using SSL port forwarding to a Tor hidden service. Note that this can be a bit of a slowdown, which might be problematic for the user who's more ambitious about number of coinjoin counterparties.

Note on Network Error Robustness
===
The code uses an out-of-the-box IRC handling class that comes with Twisted, which handles many things, e.g. reconnecting attempts with exponential backoff and various IRC protocol messages. However a number of customisations are included, e.g. support for connections to hidden services, and a specific way of handling nick conflicts (since the Joinmarket messaging protocol relies on the nick).
In practice this has gotten better over time and crashes of the long-running Maker bots are now very rare.


Clarifying the Distinction between the Maker and Taker Roles
===========================================================

The Taker pays for "liquidity" in coinjoins, but it's more than that.

The Taker's advantages are: (1) getting a coinjoin immediately, not waiting, (2) choosing the exact bitcoin amount of the coinjoin (equal-sized) outputs, (3) choosing the list and exact number of counterparties (they can even choose one-by-one with the "-P" command) based on any criteria they like, (4) being the constructor of the transaction, and so being the only one who knows the linkages of the *other* counterparties, i.e. none of the other counterparties (the Makers) know which output is theirs, unless they *all* collude; (5) they take less security risk as they do not need to have an always-on hot wallet over many weeks, months etc. to achieve privacy improvement.

The Maker's advantages are: (1) they get paid, in an anonymous and trust-free way, bitcoins and, (2) they get (not guaranteed) privacy advantage even if in individual transactions, the Taker knows their linkage, overall, over many coinjoins it's unlikely that the full linkages are known.


Common Elements between Maker and Taker Bots/Agents
===================================================
Both bots use the 'client-server' design as described above and of course have to talk the same Joinmarket protocol on the message channels. They also both keep an instance of `OrderbookWatch` which stores economic data about coinjoin offers in a sqlite database in memory (generally of course Makers don't use this data except for unusual situations). The database is maintained by having a hook to both public and private message events in the message channel. They also share all bitcoin related code, of course.
On starting up the twisted reactor, the client (as described above in detail) exchanges startup info with the messaging daemon,for all types of bots. This sets the nick and the choice of message channels and connection parameters, and prepares for all messages to be signed and verified.

How the Maker operates under the hood
=====================================
After startup, the Maker first publishes an offer in the message channel (sometimes called the 'pit', as in trading pit), which can be read by any reading bot (an orderbook watcher bot or a tumbler bot etc.), which can then update its inbuilt database to include this offer.
From there the bot waits indefinitely for private messages from Taker bots who want to do coinjoins. The JMP doc describes the exact messaging sequence, however at a high level, the JMDaemonServerProtocol instance created has its role set to "MAKER", and will respond to messages as follows:

On receiving an !orderbook request from anyone in PM, the bot will respond with its current offer list.
On receiving a !fill request from a particular bot, it will *reset* its state with that bot (so any previous negotiation is immediately abandoned). A new DH e2e encryption is set up for processing, and PoDLE commitment is checked before passing across any data. Assuming this passes, it continues the coinjoin negotiation, passing across utxos etc.(under encryption).
This can happen in parallel with an arbitrary number of Taker bots.
When it comes to conflicting transactions in parallel, a Maker will *not* disallow two Takers to arrange coinjoins with the same utxo as long as a spend is not yet confirmed. This is to prevent a denial of service by simply 'claiming' utxos transaction proposals that are then double spent out. This design can lead to transactions failing to confirm due to conflict; see more on this in "Taker" section below, but for the Maker the confirmation will time out (by default after 6 hours) and the state of the utxos will not be updated.

See the notes in `Maker.verify_unsigned_tx` ([here](https://github.com/JoinMarket-Org/joinmarket-clientserver/blob/master/jmclient/jmclient/maker.py#L172) and all the comments in this function)  w.r.t transaction signing to understand the checks undertaken to ensure that there is no possibility of signing a transaction which pays out to the Maker less than expected (input + coinjoin fee - any network fee contrib.).

The Maker uses transaction monitoring loops in `jmclient/blockchaininterface.py` (see `add_tx_notify`) to understand when to update its wallet state and then *may* (but often doesn't) announce a new offer to the trading pit for a new amount.

Offer amounts are somewhat randomised (for privacy purposes) in the most commonly used maker bot (`scripts/yg-privacyenhanced.py`).

How the Taker Operates under the Hood
=====================================
Takers are fundamentally serial rather than parallel in their operation. The taker's state is maintained in the daemon, that is to say: messages must occur in a particular sequence during the coordination of a specific coinjoin; if they come out of sequence locally an assertion fails, if out of sequence or completely irrelevant messages come from counterparties they are simply ignored (although there is a `send_error` function to communicate a failure to the counterparty, it is only used in certain cases).

After initially publishing an `!orderbook` request in public in the pit, they wait for some time to receive all offers directly from active makers via private message, and then use some algorithm to choose those with the appropriate offer sizes and fees. They then send `!fill` requests via PM to those specific makers they've chosen, along with a PoDLE commitment token. If the commitment is acceptable, encrypted messaging is established and they send the opening of the commitment (containing a pubkey and corresponding utxo) under encryption, and only then do the Makers resspond with the list of the utxos they propose to do the transaction with.
At this point the Taker can construct the transaction. Aberrant makers may fail to send utxos, or send invalid ones; in this case, the Taker will fallback to create a smaller coinjoin transaction including the Makers with valid data; but will not go lower than the value of `minimum_makers` in the config file. This gives a defence against protocol jamming, but also note it requires having a timeout set for how long we're prepared to wait if no data is sent (set in the config variable `maker_timeout_sec`).
Once the tx is constructed (and this has less security risk than in the Maker case above, since the Taker is only signing a transaction that he himself constructs), it is sent in serialized form under encryption to each Maker. The makers send back one or more signatures on the transaction with `!sig` messages (in parallel of course), as before the same timeout is used to give the time needed for this to occur.
If there is a similar "protocol jamming" at this stage, it's more problematic; since the exact ins/outs of the transaction are already set. Therefore, if this stage times out without a full set of signatures, the Taker has no choice but to restart the entire transaction negotiation, but he does so with the set of Makers that responded correctly in the first place. A similar design is seen in e.g. Coinshuffle where the protocol is restarted after a blame phase (there is no blame phase needed here, since the Taker already has the ephemeral identity of the aberrant Maker).
Once the transaction has been signed by all parties, the Taker is the final signer and broadcasts the transaction onto the Bitcoin network via RPC.
As for Makers, there are callbacks for the unconfirmed (seen on network) and confirmed events; they are not really used for a single coinjoin, but the latter is used for multiple coinjoins in series (see below).

Multiple Taker Transactions in Series
=====================================
The architecture gives the `Taker` object a `schedule` member variable, which stores the sequence of proposed coinjoins (these can be written into text files in csv format using the parameter -S to `sendpayment.py` although this is advanced usage which users won't generally use). Generally users will use the `tumbler.py` script or its equivalent in the GUI (see the `Multiple Join` subtab of the `Coinjoin` tab). This will set some sensible parameters for a sequence of coinjoins moving between each of the (by default 5) mixdepths/accounts in the wallet. Wait times are set to stagger the transactions and multiple destination addresses are used, these two features defend against timing correlation and amount correlation.

Wallet and Transaction Types
============================
The wallet is BIP32, BIP39 and BIP44-BIP49. It is (~)by design homogeneous in this regard, that we use all and only `3` (p2sh-p2wpkh) addresses, although there are arguments to allow other approaches. A future protocol update may switch to native segwit Bech32 addresses.
We use BIP32 more agressively than most other wallets. We allow the user to configure number of 'mixdepths' = accounts, but by default they have 5, and they tend to all get used.
Addresses in the wallet are imported as watch-only with labels corresponding to the wallet.
Wallet syncing using Bitcoin Core RPC is a little more complex than it probably should be, but we had to do a bit more than most on this front, in particular: there were times before the PoDLE commitment feature when people were being spammed with a lot of transaction proposals, and since every *request* to use an address represents an exposure of it, we mark addresses as used every time this happens. This resulted in some cases in huge gaps in some or all of the account branches. While it's pretty reliable today, it could probably do with an overhaul. The multiwallet feature is enabled and recommended in case you have a Bitcoin Core wallet with a lot of transaction data, since it gets scanned through.
Also to note on syncing, it would probably be desirable to add support for BIP157/8 in future, an absolute requirement of a full node is a little bit undesirable in my opinion.

The wallet's file persistence contains basic metadata (recently added: utxos in frozen state). They are encrypted using Argon for password hashing and AES-CBC.
Transaction metadata: all transactions are v1 and zero locktime; this is something I would like to change so that there's a theoretical bump in anon set, even though, of course, in practice there is no possibility of Joinmarket sharing an anon set with any other wallet, including other Coinjoin wallets, which have differences. RBF is not enabled at all.
We do not use BIP69, only randomised input and output lists (not cryptographically secure randomness but that seems a bit overkill); there seems to be a bit of a debate, although clearly it's not a huge deal either way.
Most importantly, to avoid fee flagging, the bitcoin network transaction fee chosen by Takers is either (a) taken from Core's `estimatesmartfee` with sensible parameters, so should have a reasonable anon set, or (b) chosen with a significant (20%) noise randomisation for "fixed" fees, which should make correlation more difficult.

