JoinMarket messaging protocol
=============================

The purpose of this document is to unambiguously define how joinmarket participants communicate. The messaging substrate may change in the future (see [this class](https://github.com/chris-belcher/joinmarket/blob/master/lib/message_channel.py#L5) ), for now it is [IRC](https://github.com/chris-belcher/joinmarket/blob/master/lib/irc.py).

All messages are split into fields. Fields are separated with a single whitespace character (more than one whitespace char is [not allowed](https://github.com/chris-belcher/joinmarket/blob/master/lib/irc.py#L210)).

Messages have format:
    
!command [[msg field 1] [msg field 2] ...] 

Messages are sent in two modes: public (broadcast to all other agents) or private (directed to a specific 
agent and not seen by others).

The first field always starts with the [command prefix](https://github.com/chris-belcher/joinmarket/blob/master/lib/irc.py#L12), currently '!' and is completed with one of the following commands:

| Command word    |      | mode |
| :---------:|:----:|:-------:|
| relorder | |public or private |
| absorder | |public or private |
| orderbook | |public |
| relorder | |public or private |
| cancel | |public |
|fill||private|
|pubkey||private|
|auth||private|
|ioauth||private|
|tx||private|
|sig||private|
|error||private|
|push||private|

Private messages not starting with a command from this list are to be explicitly rejected.
Public messages not starting with a command from this list are to be ignored.
The initial command field is followed with zero or more message fields. 

Multi-part messages
===================
An unencrypted message may optionally contain more than one command. The message is first split into sub-messages on the command prefix, then each "sub-message" is treated as a distinct message, applying the rules as listed above.

Note that this is not allowed for encrypted messages, which are only allowed to send a single command field at the start of the message. It **is** currently used for the `relorder` and `absorder` commands.

Chunks
======
Messages are split into chunks for passing over the messaging channel. This is to allow  Chunks are terminated with the chunk delimiter ';', unless it is the final chunk, which case the chunk delimiter is '~'.

Messages are split into chunks before sending over the messaging channel, and the decision for chunk size
is set globally as a function of the messaging implementation (currently: [approx 400 characters for IRC](https://github.com/chris-belcher/joinmarket/blob/master/lib/irc.py#L11)).

Encryption
==========

Public messages (broadcast to all other participants) are never encrypted.

Private messages of command-type 'fill', 'pubkey' and 'error', 'orderbook, 'push, 'relorder' and 'absorder' are sent in plaintext.
Messages of command-type 'ioauth', 'auth', 'tx' and 'sig' are sent encrypted.

These rules are currently enforced [here](https://github.com/chris-belcher/joinmarket/blob/master/lib/irc.py#L15-L16) (TODO: although this should be moved to the message_channel class).

All encrypted messages are [base64 encoded](https://github.com/chris-belcher/joinmarket/blob/master/lib/enc_wrapper.py#L63-L69) before being passed onto the message channel.

For encrypted messages, the entire set of message fields are sent as a single encrypted block (including the whitespace delimiters between the message fields). The command field and the chunk indicator field are sent in plaintext. (TODO: to improve privacy padding should be added to some or all of these messages; eg, to combat MITM correlation of ioauth and sig messages with the inputs belonging to a liquidity provider, dummy inputs and signatures can be sent).

For clarification, the sequence for sending of encrypted messages is therefore plaintext-->encryption-->base64encoding-->chunking--> prepend !command to first chunk and add chunk delimiters -->send to message channel (private message for encryption). And the reverse for receiving.

Valid conversation sequences:
=========================

| TAKER    |      | MAKER |
| :---------:|:----:|:-------:|
|| <<<| ![rel\|abs]order (public)|

| TAKER    |      | MAKER |
| :---------:|:----:|:-------:|
|!orderbook(public)|>>>||
|| <<<| ![rel\|abs]order (private)|

| TAKER    |      | MAKER |
| :---------:|:----:|:-------:|
|!fill(private)|>>>||
||<<<|!pubkey(private)|
|!ioauth(private)|>>>||
||<<<|!auth(private)|
|!tx(private)|>>>||
||<<<|!sig(private)|

Each taker may speak to many makers simultaneously and each maker may speak to many takers simultaneously in private.

This document is not yet complete. Edit proposals welcomed.
