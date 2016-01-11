# Joinmarket design

## Purpose of this document

This is a *high level* description of JoinMarket's design. The goal is to allow developers and other interested parties to either develop for or work with JoinMarket as a system. As an example, it might allow a developer to create a new implementation with an entirely different codebase.

Over time, it is hoped that this will be greatly extended - in particular, via linking to sub-documents which give more details on the protocol, such as message formats etc. For example, there is already [this](https://github.com/Joinmarket-Org/joinmarket/wiki/Messaging-Protocol.md).

**Table of Contents**

* [Wallets](#wallets)

* [Transactions](#transactions)

* [Entities](#entities)

 * [Maker](#maker)

 * [Taker](#taker)


# Wallets

Joinmarket wallets are of the [BIP32 Hierarchical Deterministic wallet](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki) form, starting from the root m/0 (here '0' marks the default account, in accordance with the BIP spec). The branches are defined by:

    m/0/mixdepth/[external/internal]

The value of mixdepth runs from 0..M-1, where M is the number of mixdepths chosen by the user; by default 5. The value of [external/internal] is 0 for external, 1 for internal. Thus a default wallet will contain 10 separate branches.

Typical output from script `wallet-tool.py`, which enables basic wallet handling, running against a testnet wallet seed:

    mixing depth 0 m/0/0/
     external addresses m/0/0/0/
      m/0/0/0/016 muijch3wHvCXm9pHJQiyQFWDZSngPUdLrB  new 0.00000000 btc 
      m/0/0/0/017 n35T2GFV2CQXuN2eZJWBG7GTkhJZs4swJD  new 0.00000000 btc 
      m/0/0/0/018 mgwSKYK1zssnJi5T9UaR1gxdzPmU6zobvk  new 0.00000000 btc 
      m/0/0/0/019 mmXouP3x92h5yvBLMn6So4BvTRShN318Za  new 0.00000000 btc 
      m/0/0/0/020 mhKpprCpmfnsS2AwBDjiRoArNjSRMKCXHL  new 0.00000000 btc 
      m/0/0/0/021 n1h6MYPJLsoJ1dz2HNGxM7L5ct4A84RvQW  new 0.00000000 btc 
     internal addresses m/0/0/1/
      m/0/0/1/048 mrytep7Ne1t2Epzrn1zWPrBAfaDWCtekqx used 4.87792833 btc 
    for mixdepth=0 balance=4.87792833btc
    mixing depth 1 m/0/1/
     external addresses m/0/1/0/
      m/0/1/0/045 mrjkGaHdvFqZiTXRWcvAqSTmLaxKRfDWB9 used 9.27910795 btc 
      m/0/1/0/046 mmMs5iHLZKaLHLG3zysHwEdBNVHGyCLXgf  new 0.00000000 btc 
      m/0/1/0/047 mfkPoc34Z2k15jhcwPc8ifywykXvkovv9z  new 0.00000000 btc 
      m/0/1/0/048 mn5AuW8due3KtTGXvi68xQQUyNkg8jo5uQ  new 0.00000000 btc 
      m/0/1/0/049 n4Wg3ajPgLvEpumK2sHZXiggUw3kX6XCSh  new 0.00000000 btc 
      m/0/1/0/050 mfnxp5n4nMa9fZogNBo5gTSA7HBRLYmCW2  new 0.00000000 btc 
      m/0/1/0/051 msRJ7MpC9gKjqi7GjJ8WrHqAnPwDT8Nxsc  new 0.00000000 btc 
     internal addresses m/0/1/1/
      m/0/1/1/043 mtBYHv4vmxKfqQT2Cr2vrA578EmUzcwwKA used 1.76456658 btc 
      m/0/1/1/044 mp9yvRWwu5Lcs5EAnvGSKMKWith2qv1Rxt used 4.58622784 btc 
    for mixdepth=1 balance=15.62990237btc
    mixing depth 2 m/0/2/
     external addresses m/0/2/0/
      m/0/2/0/042 muaApeqh9L4aQvR6Fn52oDiqz8jKKu9Rfz  new 0.00000000 btc 
      m/0/2/0/043 mqdZS695VHeNpk83YtBmutGFNcJsPC39wW  new 0.00000000 btc 
      m/0/2/0/044 mq32QZX7DszZCYzKP6TvZRom1xFMoUWkbS  new 0.00000000 btc 
      m/0/2/0/045 mterWqVTyj2oKHPr1raaY6iwqQTKbwM5ro  new 0.00000000 btc 
      m/0/2/0/046 mzaotzzCHvoZxs2RjYXWN2Yh9kB6F5gU8j  new 0.00000000 btc 
      m/0/2/0/047 mvJMCGoG5ZUARtAr5x2KD1ebNLuL618zXU  new 0.00000000 btc 
     internal addresses m/0/2/1/
    for mixdepth=2 balance=0.00000000btc
    mixing depth 3 m/0/3/
     external addresses m/0/3/0/
      m/0/3/0/002 mwjZyUbh78UndNtqsKathxpjE4EsYrhLzm used 3.00000000 btc 
      m/0/3/0/004 msMoHnZRKfNRPQqg2MUk1rXrqwM6VeLL8C used 0.30000000 btc 
      m/0/3/0/006 mhBHRaFwMTPPWLdukqgJaSMmmCeQ8ef6QV used 9.26824652 btc 
      m/0/3/0/007 mrdv38dX1eQsZjAof442G9Lj66bM5yUnjA used 4.58134572 btc 
      m/0/3/0/008 moWA3FEYkNbEJkHdnQbQYkrVztTviMaHwe used 5.39567085 btc 
      m/0/3/0/009 mgoTGzD46mWoMiH97QHh9TxuJp4NmVobNp used 3.00000000 btc 
      m/0/3/0/010 mjddF8HmBaenGBspTMzhF3qikbbLB4xGZN  new 0.00000000 btc 
      m/0/3/0/011 mnEvKc2JLj8s1GLfvtvqVt5pWTu5jCdtEk  new 0.00000000 btc 
      m/0/3/0/012 n1FwKgDEjzRj2YsZNoG5MWVnuq72wA8Mgr  new 0.00000000 btc 
      m/0/3/0/013 mq4vn4KX9SRrvhiRkg9KrYADhKNPAp2wyN  new 0.00000000 btc 
      m/0/3/0/014 n3nG4334F19THUACgEEveRX8JcE5awY1qz  new 0.00000000 btc 
      m/0/3/0/015 mz5yHHbud68b8CLeNk2MsnyFgSJ9rPGk7v  new 0.00000000 btc 
     internal addresses m/0/3/1/
      m/0/3/1/000 mfbeMyajM4wYRmpWVrYnd3rBqvtBniw8gj used 0.25672994 btc 
      m/0/3/1/001 mzkFk3C9J6D9KE9cV4kuSctUKZXkr1gaio used 0.51499000 btc 
      m/0/3/1/002 n3SCTwDs4wcn7yGhFq8HJwxQ9PzBXvTJdV used 0.04199000 btc 
      m/0/3/1/006 mow4CyHNssjo2CNHdg3DdbXWYNPfY8HEPe used 0.36073270 btc 
    for mixdepth=3 balance=26.71970573btc
    mixing depth 4 m/0/4/
     external addresses m/0/4/0/
      m/0/4/0/011 mzMWwUSvj4n3wgtSMLXt17hQTi2zinzYWZ used 2.00000000 btc 
      m/0/4/0/012 mrgNjQWjrBDB821o1Q3qG6EmKJmEqgjtqY  new 0.00000000 btc 
      m/0/4/0/013 msjynFwGoGePoheVABAEgjxjw8FNuPgtPC  new 0.00000000 btc 
      m/0/4/0/014 moJpd6ZvgAm1ZENmAuoPKnqAUntoSixao9  new 0.00000000 btc 
      m/0/4/0/015 muajyWxmjuHQDm3MpSr7Y3RLAWDqch1yix  new 0.00000000 btc 
      m/0/4/0/016 n2tPijSN4nBFGAKbXpp6H7hfznhsYWwR88  new 0.00000000 btc 
      m/0/4/0/017 mnkrhe7fMYy4yjPM54gEurgxWpDXJ3axtn  new 0.00000000 btc 
     internal addresses m/0/4/1/
    for mixdepth=4 balance=2.00000000btc
    total balance = 49.22753643btc

## Use of external and internal branches.

Payments into the wallet should be made into new addresses on the `external` branch for any mixdepth. For the above wallet, `muaApeqh9L4aQvR6Fn52oDiqz8jKKu9Rfz` (from mixdepth 2) or `mrgNjQWjrBDB821o1Q3qG6EmKJmEqgjtqY` (from mixdepth 4) would be suitable candidates. The index of the address on the branch is shown as the final 3 digit integer in the identifier. As usual with deterministic wallets, a configurable gap-limit variable is used to determine how far forwards to search after unused/new addresses are located.

In a joinmarket [transaction](#transactions), outputs go to: (1) coinjoin outputs -> external addresses in the next mixdepth (where 'next' means (mixdepth+1 % M)), (2) change outputs -> internal addresses in the same mixdepth.

The logic is fairly straightforward: the coinjoin outputs of a transaction must not be reused with any of the inputs to that same transaction, or any other output that can be connected with them, as this would allow fairly trivial linkage. By moving coinjoin outputs to an entirely separate branch, isolation is enforced.

## Wallet generation and access control

The master private key is generated via a call to Python's `os.urandom`, being the interface to the underlying OS randomness source. It is a 32 byte random string.

This seed is used to generate the wallet structure described above and according to the BIP32 specification.

The recovery seedphrase is a 12 word phrase of the type used in earlier versions of Electrum, and using that code base, specifically see the functions `mn_encode` and mn_decode` in the module `old_mnemonic.py`, which uses a 1626 word list, also found in that file (note: 16**12 implies 128 bit security). A commonly asked question is whether and why not Joinmarket supports BIP39. For now, the answer to that question is that it isn't relevant; since JM's wallet uses this specific HD structure, designed to allow coinjoins to occur safely, it is not directly compatible with other wallets. This may change in the future, however.

The wallet seed is encrypted for persistent storage using AES in CBC mode, using the module `slowaes.py`. Note that a bug was found earlier in this module's handling of PKCS7, which could have allowed 'decryption' to garbage, but this was [fixed](https://github.com/JoinMarket-Org/joinmarket/pull/191).

The wallet is persisted to disk in this format:

    {"index_cache": [[113, 163], [149, 161], [154, 134], [128, 149], [135, 120]], "encrypted_seed":
    "15336f8220ee6da168c153e1e11c0402c862fa377d2422c27b5f92ed37d52a840d1a80c4eda7ca9731ec5e0ebfa92cdd",
    "creation_time": "2015/05/08 15:56:58", "network": "mainnet", "creator": "joinmarket project"}

**index_cache** is a list of 5 x 2 entries, one for each branch in the wallet (in the default case) as described above. Each number is a pointer to the next unused address on the branch. This is of high importance because it allows the entity in control of the wallet to ensure that an address on that branch is not reused in more than one transaction. To achieve this, it's necessary that the function `Wallet.update_index_cache` is called *immediately* a new transaction is *proposed* for that entity, even if the transaction fails to complete. Note that this can lead to practical difficulties (such as large wallet gaps) in the case of Sybil attacks or errors where a long string of proposed but incompleted transactions occurs.

**encrypted_seed** is a 48 byte hex encoded string - this is 3 AES blocks, the plaintext being the 32 byte seed, the final block's plaintext being entirely pkcs7 padding.

**creation_time**, **network** and **creator** are self-explanatory.

## The `wallet-tool.py` script

This script is currently (may change after GUI tools are developed) the way a user can perform two functions: (a) generating, recovering and funding a wallet, and (b) viewing a wallet's status. See `--help` for an explanation of its features.

# Transactions

## Core concepts

The basic idea of CoinJoin is explained in detail [here](https://bitcointalk.org/index.php?topic=279249.msg2983902#msg2983902). JoinMarket transactions are of the simplest type described there: there is one coordinating party (the *taker*) who combines his own inputs with inputs from many *makers* and produces a template of a transaction with N+1 **equal** outputs (of a size decided by the taker), one for each maker plus himself, and as many change outputs as are required. In the current Joinmarket implementation, there is one change for each participant, except (a) if the taker has deliberately chosen to spend all of his inputs to the coinjoin output, see [sweeps](#sweepjmtx), or (b) in rare cases, by accident, one maker has exactly the right amount in his inputs for the coinjoin output.

After constructing the template transaction, the taker broadcasts the template to the makers, who transfer valid signatures back to the taker. The taker then adds his own signatures and broadcasts the bitcoin transaction.

This design allows for change outputs for maximum practicability. Insisting on *only* equal sized outputs (only "coinjoin outputs" in this terminology) would considerably improve privacy benefits but makes coordinating to achieve the goal nearly impossible.

Critical to making this work effectively is avoiding **address reuse**. If coinjoin outputs later get reused with change outputs that link back to either the inputs, or the change outputs, in the same transaction, then the coinjoin output can be linked to them, destroying the purpose of the transaction. For this reason, Joinmarket [wallets](#wallets) use a BIP32 hierarchical deterministic structure and enforce that the coinjoin outputs are on a different branch than the inputs and change outputs.

The next section shows the basic types of transaction that can occur in JoinMarket and illustrate these points in a more concrete fashion.

## Joinmarket transaction types

We can define 4 distinct joinmarket transaction types. We use the notation "JMTx" for all of these, and separate short hand versions for each one.

1. SourceJMTx
 This is usually an ordinary bitcoin transaction paying into an unused address on an external branch for any one of the mixdepths of the wallet. As such it has no special joinmarket structure; it will usually have a change output, which will go back to the wallet funding this one. It *could*, however, be a payment from another joinmarket wallet, although most users will not be using more than one joinmarket wallet. This doesn't affect the analysis, in any case.

2. (Canonical) CJMTx
 The most fundamental type of joinmarket transaction involves neither a 'source' nor a 'sink', but only spends from this joinmarket wallet to itself, in conjunction with joining counterparties. The coinjoin output goes to a new address on the external branch of the *next* mixdepth, as was described in the previous section.

 ![alt text](/images/CJMTx.png)

 **NOTE**: the above diagram ignores transaction fees and coinjoin fees for simplicity, which slightly change the size of the change outputs; this obviously cannot be ignored in real analysis.

3. SpendJMTx
 This is a 'sink' transaction. This, generated either by `sendpayment.py` or `tumbler.py` scripts usually, will be the same as the CJMTx above, except with the difference that the coinjoin output for the initiator (taker) goes to an external address, e.g. a payment or a transfer to an external wallet.

4. SweepJMTx
 This can be either a 'sink' or not, depending on the destination of the coinjoin output. A sweep is characterised by 2 things: (a) the initiator will not create a change output, and (b) the initiator will consume **all** of the utxos in the given mixdepth (both internal and external branches) as inputs, leaving no coins left in that mixdepth.

 ![alt text](/images/SweepJMTx.png)

# Entities

All entities are subclassed from `CoinJoinerPeer`, which has the instance member `self.msgchan`. This is an instance of class `MessageChannel`, which abstracts the functionality required for the peer to participate in the communications channel.

There are currently three types of `CoinJoinerPeer`:

1. `OrderbookWatch`
2. `Taker`
3. `Maker`

`OrderbookWatch` does not participate in the Joinmarket protocol, but is an abstraction to allow observation of the Joinmarket pit. It stores order information in a sqlite3 db object.

`Taker` extends from `OrderbookWatch`, inheriting the db object for querying orders in the joinmarket pit, allowing the taker to make decisions about which orders to fill. Maker inherits directly from `CoinJoinerPeer` since it does not need this information.

The required functionality of `Taker` and `Maker` objects is described in the succeeding sections.

---

## Maker

Existing instances: `yield-generator-basic.py`, `yield-generator-mixdepth.py`, `yield-generator-deluxe.py`, `patientsendpayment.py`

A Joinmarket maker offers liquidity for coinjoins (hence the terminology, borrowed from financial exchanges - *liquidity maker*). At a high level, it offers this functionality:

* Enter the joinmarket trading pit, using the functionality provided by the `self.msgchan` instance variable.
* Announce information about the parameters for coinjoins it is willing to carry out - amounts, fees.
* Wait until a `Taker` requests a coinjoin
* Participate in a handshake to set up the coinjoin transaction
* Provide the `Taker` with signatures for an agreed upon coinjoin transaction
* Watch the blockchain for updates, updating its state (including wallet) when a transaction appears.

### Design considerations

As can be seen, a maker offers a service, so after the message channel is set up, its methods get triggered by callbacks from the main listening loop in the messaging channel. The startup process is, at a high level, : 

* Parse command line and load configuration from `joinmarket.cfg`
* load wallet
* initialize message channel and register callbacks
* start message channel listening loop

Once a request is received from a taker, the maker creates a new `CoinJoinOrder` object instance, and this performs the handshake, initializes encryption on the messaging channel, and exchanges the data for the transaction. Then, a listening thread is set up to listen for **announcement of the new bitcoin transaction on the network** (see `unconfirm_callback`), and for **inclusion of the bitcoin transaction in a block** (see `confirm_callback`).

**NB: Handling concurrent requests:** If two takers attempt to fill the same order at once, the maker will **not** lock the utxos immediately a request is initiated, but will mark those utxos as consumed as soon as the `unconfirm_callback` is triggered (i.e. the negotiated transaction is seen on the bitcoin network). A little reflection will show that this is the only design that makes sense. After a taker has negotiated a transaction, he is under no obligation to actually broadcast it to the bitcoin network, nor is he certain to have received the right signatures from all other makers (they may have crashed, or dropped off the network, or sent garbage signatures, etc.). So this maker has no particular reason to expect that the transaction will ever be broadcast. Even worse, a taker may deliberately DOS attack makers by requesting transactions and then not broadcasting. Hence, it's necessary that makers do not mark utxos as "used" until and unless they see the negotiated transaction broadcast. This *does* create a slightly unfortunate fact that two takers who successfully negotiate different transactions using this maker's utxos are in a race to broadcast them, but this is unavoidable.

**Keeping track of state for individual takers**: A maker will only perform one transaction for one particular taker at a time. The instance variable `self.active_orders` is a dict for which the keys are the names of taker counterparties, and the values are `CoinJoinOrder` objects, each of which contains a `libnacl.public.Box` object used for encrypted messaging with that counterparty. If a taker tries to start a new transaction while a previous one is incomplete, the new `CoinJoinOrder` overwrites the old one (and a new encryption Box is set up).

### The CoinJoinOrder object

The `CoinJoinOrder` object is perhaps poorly named; it manages the process of negotiating a single transaction with a taker for this maker, once a order fill request has been received. See [here](https://github.com/Joinmarket-Org/joinmarket/encryption_protocol.txt) for the steps of negotiation.

If the negotiation is successful from the maker's point of view, it will add a notification function to the list `txnotify_fun` in the `BlockchainInterface` instance, and thus register callbacks `unconfirm_callback` and `confirm_callback` for this transaction.

### Unconfirm and confirm callbacks for a `CoinJoinOrder` object

When an unconfirmed transaction is seen on the network (as justified above), the wallet state must be updated to remove those utxos, and the orders that the maker is offering should change. This is done by calling `Maker.modify_orders` and passing the delta to existing orders, which is calculated by the algorithm the maker is using (so, this is a function of the maker's algorithm). That delta is found by calling `Maker.on_tx_confirmed` and `Maker.on_tx_unconfirmed` - note that these methods will be overriden by subclasses (at the moment, `YieldGenerator`). Utxos will also be added, of course, in calling `on_tx_confirmed` - the newly created ones.

---

## Taker

Existing instances: `sendpayment.py`, `tumbler.py`, `patientsendpayment.py`

A joinmarket taker chooses a set of makers, based on examination of the orderbook, and initiates a coinjoin transaction, and broadcasts it to the network. It may choose to do this repeatedly. At a high level, its functionality looks like this:

* Enter the joinmarket trading pit, using the functionality provided by the `self.msgchan` instance variable.
* Request the orderbook and store this information in its `self.db` instance variable.
* Decide which subset of orders it wants to fill based on amounts, fees and its own configuration/algorithm.
* Initiate the handshake and transaction data transfer with the chosen set of makers.
* Create the template of the intended coinjoin transaction and send to the makers.
* Wait until it has received all the signatures for the agreed upon coinjoin transaction
* Serialize the final transaction and broadcast it to the bitcoin blockchain via the `BlockchainInterface`, then:
 * Quit, OR
 * Watch the blockchain for updates, updating its state (including wallet) when a transaction appears, then:
 * loop back to 'request the orderbook' above, in order to start the next transaction in the series.

### Design considerations

Although a taker is a *consumer* of service rather than a provider, nevertheless it has the same basic architecture as a maker: after the message channel is set up, its methods get triggered by callbacks from the main listening loop in the messaging channel. So the startup process is basically identical to that listed above for the maker.

Whether the taker initiates one transaction or many, once it has broadcast its final transaction, it simply quits. For a series of transaction, it uses the same design as the maker to listen for blockchain events: it adds `unconfirm_callback` and `confirm_callback` to its `BlockchainInterface` instance. Since there is no competition for its utxos, there is no need for specific actions in response to transaction arrival on the bitcoin network, so `unconfirm_callback` need not do anything (except perhaps print debugging information), but the `confirm_callback` can be used to trigger the next transaction in the taker's chosen sequence of transactions, and also requires update of the wallet state to refer to the new utxos.

### The `CoinJoinTX` object

This plays a similar role to `CoinJoinOrder` for `Maker`. A taker only processes one `CoinJoinTx` object at a time, and its process is initiated in the constructor. When a taker is ready to initiate a transaction with a decided-on set of makers, it calls `Taker.start_cj()` with arguments specifying its chosen orders (from makers), coinjoin amount, output addresses, utxos and transaction fee and then initiates a handshake, sets up encryption, transfers transaction data, receives signatures and then signs itself and pushes the transaction to the bitcoin network.

### Handling unresponsive makers

It is not uncommon for the makers to fail to respond at any stage of transaction negotiation. To handle this, the `CoinJoinTX` object contains a `TimeoutThread` which keeps track of how long we've been waiting for all of the makers specified for the transaction. If the timeout (specified in the configuration variable `maker_timeout_sec`) is exceeded, the function `CoinJoinTX.recover_from_nonrespondants()` is called, allowing the taker to restart the process from the beginning.

---

