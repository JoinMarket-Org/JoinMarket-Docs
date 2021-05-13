JoinMarket messaging protocol
=============================

The purpose of this document is to unambiguously define how joinmarket participants communicate. The messaging substrate allows for communication over multiple `MessageChannel` objects, managed by a single `MessageChannelCollection` object; see [this module](https://github.com/JoinMarket-Org/joinmarket/blob/master/joinmarket/message_channel.py).

All messages are split into fields. Fields are currently separated with a single whitespace character (more than one whitespace char is not allowed.

Messages have format:
    
!command [[msg field 1] [msg field 2] ...] 

Messages are sent in two modes: public (broadcast to all other agents) or private (directed to a specific 
agent and not seen by others).

The first field always starts with the [command prefix](https://github.com/JoinMarket-Org/joinmarket/blob/35dc60848c201f1c071a00c885969d1cc458cbbf/joinmarket/message_channel.py#L8), currently '!' and is completed with one of the following commands:

| Command word    |      | mode |
| :---------:|:----:|:-------:|
| reloffer | |public or private |
| absoffer | |public or private |
| orderbook | |public |
| cancel | |public |
| hp2 | |public |
|fill||private|
|pubkey||private|
|auth||private|
|ioauth||private|
|tx||private|
|sig||private|
|error||private|
|push||private|
|tbond||private|

Private messages not starting with a command from this list are to be explicitly rejected.
Public messages not starting with a command from this list are to be ignored.
The initial command field is followed with zero or more message fields. 

Multi-part messages
===================
An unencrypted message may optionally contain more than one command. The message is first split into sub-messages on the command prefix, then each "sub-message" is treated as a distinct message, applying the rules as listed above.

Note that this is not allowed for encrypted messages, which are only allowed to send a single command field at the start of the message. It **is** currently used for the `reloffer` and `absoffer` commands.

IRC-specific feature: chunks
======
Messages are split into chunks for passing over an IRC messaging channel; this code is found therefore in the [irc module](https://github.com/JoinMarket-Org/joinmarket/blob/35dc60848c201f1c071a00c885969d1cc458cbbf/joinmarket/irc.py#L236-L252). Chunks are terminated with the chunk delimiter ';', unless it is the final chunk, which case the chunk delimiter is '~'.

Messages are split into chunks before sending over the messaging channel, and the decision for chunk size
is set globally as a function of the messaging implementation (currently: [450 characters](https://github.com/JoinMarket-Org/joinmarket/blob/35dc60848c201f1c071a00c885969d1cc458cbbf/joinmarket/irc.py#L18)).

Future non-IRC message channel implementations thus may or may not use this feature, depending on their characteristics, but all IRC implementations must use it.

Encryption
==========

Public messages (broadcast to all other participants) are never encrypted.

Private messages of command-type `fill`, `pubkey`, `error`, `orderbook`, `push`, `reloffer`, `absoffer` and `tbond` are sent in plaintext.
Messages of command-type `ioauth`, `auth`, `tx` and `sig` are sent encrypted.
These rules are enforced [here](https://github.com/JoinMarket-Org/joinmarket/blob/35dc60848c201f1c071a00c885969d1cc458cbbf/joinmarket/message_channel.py#L13-L16) and [here](https://github.com/JoinMarket-Org/joinmarket/blob/35dc60848c201f1c071a00c885969d1cc458cbbf/joinmarket/message_channel.py#L818-L826).

All encrypted messages are [base64 encoded](https://github.com/JoinMarket-Org/joinmarket/blob/35dc60848c201f1c071a00c885969d1cc458cbbf/joinmarket/enc_wrapper.py#L91-L99) before being passed onto the message channel.

For encrypted messages, the entire set of message fields are sent as a single encrypted block (including the whitespace delimiters between the message fields). The command field and the chunk indicator field (for IRC) are sent in plaintext. (TODO: [pad messages to improve privacy](https://github.com/JoinMarket-Org/joinmarket/issues/352); eg, to combat MITM correlation of ioauth and sig messages with the inputs belonging to a liquidity provider).

For clarification, the sequence for sending of encrypted messages is therefore plaintext-->encryption-->base64encoding-->chunking--> prepend !command to first chunk and add chunk delimiters -->send to message channel (private message for encryption). And the reverse for receiving.

For multiple message channels: message signatures for anti-spoofing
=======================

##### Pubkey-based nicks
The username/nick/nym for the Joinmarket bot is ephemeral-per-session, but is constructed in such a way that it can be signed for, so that it is not possibly to successfully spoof that user by registering the same nick on another active message channel. The nick is constructed as follows:

A 32 byte private key is generated at random.

A Bitcoin secp256k1 public key is [created](https://github.com/JoinMarket-Org/joinmarket/blob/35dc60848c201f1c071a00c885969d1cc458cbbf/joinmarket/message_channel.py#L114) using it.

nick = one "type" byte (currently "J") + one version byte (current `JM_VERSION` protocol value, [here](https://github.com/JoinMarket-Org/joinmarket/blob/35dc60848c201f1c071a00c885969d1cc458cbbf/joinmarket/configure.py#L55)) + Base58 (not Base58Check) of: first `joinmarket.message_channel.NICK_HASH_LEN` bytes of sha256 of : ephemeral per-bot-process public key as just described.

If length(nick) < `joinmarket.message_channel.NICK_MAX_ENCODED`, right pad with 'O' char to that length.

Thus according to current rules the nick is 16 characters in length, consisting of one type byte, one version byte and 14 bytes of pubkey-hash (right padded if necessary to fix length).

##### Applying signatures to private messages.
**All** private messages (whether encrypted or not) are extended with the additional fields <pubkey> <signature> where pubkey is the Bitcoin pubkey mentioned above, hex encoded, and the signature is a Bitcoin signature, also hex encoded (TODO: this could be substantially improved in encoding size with pubkey recovery, and more compact encoding). See [here](https://github.com/JoinMarket-Org/joinmarket/blob/35dc60848c201f1c071a00c885969d1cc458cbbf/joinmarket/message_channel.py#L851-L858).

As defence against replaying the same signature on different message channels, note specifically that the plaintext which is signed is `message+hostid`, see [here](https://github.com/JoinMarket-Org/joinmarket/blob/35dc60848c201f1c071a00c885969d1cc458cbbf/joinmarket/message_channel.py#L855), where `hostid` is the hostid of this specific MessageChannel object, for IRC this is the "network" field passed from the IRC server.

Note that since this extension to the message occurs in the abstract message class MessageChannel, any chunking as described above occurs after it. The `pubkey` and `sig` are just 2 additional fields for any private message sent.

All received private messages are verified against the nick of the sender [here](https://github.com/JoinMarket-Org/joinmarket/blob/35dc60848c201f1c071a00c885969d1cc458cbbf/joinmarket/message_channel.py#L932-L939) to prevent the threat of spoofing nicks.

Valid conversation sequences:
=========================

(Announcements, not interactive/conversation):

| TAKER    |      | MAKER |
| :---------:|:----:|:-------:|
|| <<<| ![rel\|abs]offer (public)|
||<<<| !cancel(public)|
||<<<| !hp2(public)|

| TAKER    |      | MAKER |
| :---------:|:----:|:-------:|
|!orderbook(public)|>>>||
|| <<<| ![rel\|abs]order (private)|
|| <<<| !tbond (private)|

| TAKER    |      | MAKER |
| :---------:|:----:|:-------:|
|!fill(private)|>>>||
||<<<|!pubkey(private)|
|!auth(private)|>>>||
||<<<|!ioauth(private)|
|!tx(private)|>>>||
||<<<|!sig(private)|


Each taker may speak to many makers simultaneously and each maker may speak to many takers simultaneously in private.

#### Private conversation in detail:

See "Definitions" section below for the key for the abbreviations.
```
1: M: !ordertype [order details]!ordertype [orderdetails]!tbond [proof] ... (NS)
```

```
2: T: !fill order-id amount tencpubkey C (NS)
```
 
```
3: M: !pubkey mencpubkey (NS)
```

```
4: T*: !auth O (NS)
```

```
5: M*: !ioauth ulist maker_auth_pub coinjoinA changeA B(mencpubkey) (NS)
```

```
6: T*: !tx txhex (NS)
```

```
7: M*: !sig txsig (NS)
```

#### Notes:

After step 2, if provided commitment is blacklisted according to maker's policy, maker quits. Current implementation stores utxo commitments (note that there are `taker_utxo_retries` possible commitments per utxo, and that the commitments do not reveal the utxo, thus effectively the same utxo can be tried that many times) in a file named `blacklist`.

After step 4, if PoDLE verification fails, the maker quits before sending `!ioauth`, thereby disallowing receipt of owned utxos.

For PODLE calculation see [here](https://gist.github.com/AdamISZ/9cbba5e9408d23813ca8#defence-2-committing-to-a-utxo-in-publicplaintext-at-the-start-of-the-handshake)


In 5, `!ioauth ulist maker_auth_pub coinjoinA changeA B(mencpubkey) (NS)`, returned by maker, it's to be noted here that `maker_auth_pub` is needed to use an *input* , rather than an output, pubkey, as the anti-MITM key for the reason that a maker might not always own the coinjoin output key (see e.g. patientsendpayment mixing of maker/taker role).


Nick signature: currently re-calculates expected nick from given pubkey then validates signature against the entire message before NS. In future could cut out ~ 66 bytes by making use of secp256k1 pubkey recovery. The purpose of this feature is to prevent impersonation of a running bot by using its nick on a different message channel (see issue 568).

Taker side auth of utxo has been REMOVED, see overview of argument [here](https://github.com/JoinMarket-Org/joinmarket/issues/171#issuecomment-235868269) and see also [earlier background](https://github.com/JoinMarket-Org/joinmarket/pull/90#issuecomment-108648823).

#### Definitions

**T**: taker
:
**M**: maker

__\*__ : indicates message **not including** NS is encrypted

**[t,m]encpubkey** : taker and maker encryption pubkeys, per transaction, for ECDH setup

**C**: single data field for commitment to a utxo by taker, by default for PoDLE (C=H(P2)). First byte is commitment type, default "P" for PoDLE.

**B(p)**: bitcoin signature of an encryption pubkey p (signed message format standardised as for Bitcoin Core etc.)

Commitment data for default PoDLE construct:

**O**: commitment opening serialization, consists of the following fields separated by a field separator (default '|'):

 **U**: utxo used for PoDLE (not necessarily from wallet)
 
 **P**: pubkey used for PoDLE (not necessarily from wallet)
 
 **P2**: shifted base point pubkey for PoDLE

 **s, e**: schnorr sig values for PoDLE

**maker_auth\_pub**: pubkey of keypair used to authorize and setup encryption by maker (must correspond to one of maker's input addresses)

**coinjoinA**: coinjoin address used by maker

**changeA**: change address used by maker

**ulist**: list of utxos that maker proposes to be used in transaction

**(NS)**: nick signature (either of form pub, sig or from pubkey recovery, bitcoin type) : message to be signed is the whole message to be sent + message channel identifier str(`serverport`) (the latter to prevent cross-channel replay).


Fidelity Bonds
==========

JoinMarket version 0.9 implements fidelity bonds, for a full writeup of whys and hows of the feature see [the design document](https://gist.github.com/chris-belcher/18ea0e6acdb885a2bfbdee43dcd6b5af/).

Makers who have a fidelity bond will send a proof of it to takers in response to an !orderbook message along with the maker's offers.

The fidelity bond proof message contains everything needed to prove to the taker that the fidelity bond UTXO is real, that it has a certain value, and that it is genuinely possessed by the maker.
To keep the size down the fidelity bond proof message is a base64-encoded binary data blob.

The binary blob is made up of concatenating various parameters (size in bytes on the line below):

```
nick_sig + cert_sig + cert_pubkey + cert_expiry + utxo_pubkey + txid + vout + timelock
72       + 72       + 33          + 2           + 33          + 32   + 4    + 4 = 252 bytes
```

The command name "tbond" means "time locked bond".
One day there might be a burned coins fidelity bond too, but probably not because timelocked bonds have smaller proof sizes and you can get the equivalent of a burned coin bond by using a timelock very far in the future.

### UTXO data
To prove that the UTXO exists, the proof contains the `txid` and `vout` of the UTXO, these can be used to lookup in a full node's UTXO set to check that the UTXO exists, is confirmed in the best chain and is unspent.
The proof also contains a `utxo_pubkey` and `locktime` which together are enough to construct a redeemscript for a timelocked address.
The UTXO set query will give the verifier information about the UTXO's scriptPubKey and allow him to verify that the UTXO is indeed a timelocked address.
The UTXO set query will also tell the verifier how many bitcoins are locked in the address, as well as the confirmation time of the transaction.

The receiver must check that the given (`txid`, `vout`) has not already been seen. A UTXO must only be used at most once for a fidelity bond in a given orderbook.
If a UTXO does arrive at the taker a second time, the taker should just pick one (it doesnt matter which one, either way removes the incentive to announce UTXOs more than once).

### Two signatures

There are two signatures in the proof.

Diagram of the two signatures:

`Fidelity bond keypair ----signs----> certificate ----signs----> IRC nicknames`

#### Certificate signature
In order to allow holding the fidelity bond UTXOs in cold storage, there is an intermediate keypair called the certificate.
A fidelity bond privkey can sign a certificate message and transfer the `utxo_pubkey`, `cert_pubkey`, `cert_sig` and `cert_expiry` to the online computer.

The certificate message is defined as `'fidelity-bond-cert|' + cert_pubkey + '|' + cert_expiry` encoded as bytes where + denotes concatenation.

The certificate expiry `cert_expiry` is the number of the 2016-block period after which the certificate is no longer valid.
For example, if `cert_expiry` is 330 then the certificate will become invalid after block height 665280 (= 330x2016).
The `cert_expiry` is expressed as an ascii string of numerals, for example "55" for the number 55 (in hex 0x3535 or \x35\x35).
The purpose of the expiry parameter is so that in case the hot wallet really does get hacked, the hacker can only impersonate the fidelity bond for a limited amount of time and not forever.
As `cert_expiry` is stored in 2 bytes in the proof, its maximum value is 65535 which corresponds to a block height of 65535x2016 = 132118560.

The certificate signature `cert_sig`, which is the signature resulting from signing the certificate message with the fidelity bond bitcoin private key.

#### Nick signature

In order to stop reply attacks there is signature over the IRC nicknames of both the taker and maker.
This nick message is `taker_nick + | + maker_nick` encoded as bytes, for example "J54LS6YyJPoseqFS|J55VZ6U6ZyFDNeuv".

The nick signature `nick_sig` results from signing the two IRC nicknames with the certificate private key.

All signatures are ecdsa signatures in the DER format, padded at the start to be exactly 72 bytes long. The header byte of the DER format is 0x30 which makes it easy to strip the padding.




This document is not yet complete. Edit proposals welcomed.
