# Joinmarket design

## Purpose of this document

This is a *high level* description of JoinMarket's design. The goal is to allow developers and other interested parties to either develop for or work with JoinMarket as a system. As an example, it might allow a developer to create a new implementation with an entirely different codebase.

Over time, it is hoped that this will be greatly extended - in particular, via linking to sub-documents which give more details on the protocol, such as message formats etc. For example, there is already [this](https://github.com/JoinMarket-Org/JoinMarket-Docs/blob/master/Joinmarket-messaging-protocol.md).

**Table of Contents**

* [Wallets](#wallets)

 * [External and Internal Branches](#use-of-external-and-internal-branches)

 * [`Wallet` object](#wallet-object)

 * [Wallet generation and access control](#wallet-generation-and-access-control)

 * [Wallet persistence](#wallet-persistence)

 * [`wallet-tool.py`](#the-wallet-toolpy-script)

* [Transactions](#transactions)

 * [Core concepts](#core-concepts)

 * [Joinmarket transaction types](#joinmarket-transaction-types)

* [Entities](#entities)

 * [Maker](#maker)

 * [Taker](#taker)

   * [sendpayment](#the-sendpayment-script)

    * [tumbler](#the-tumbler-script)

* [Messaging Layer](#messaging-layer)

 * [Encrypted messaging](#encrypted-messaging)

 * [Messaging handshake](#messaging-handshake)

 * [Messaging protocol](#messaging-protocol)

* [Blockchain Interface](#blockchain-interface)

 * [The Notify Thread](#the-notify-thread)
* [Bitcoin Transaction Fees](#bitcoin-transaction-fees)

* [Orders and the trading pit](#orders-and-the-trading-pit)

 * [Public entity identities](#public-entity-identities)

 * [Orders](#orders)

* [The Configuration File](#the-configuration-file)


# Wallets

Joinmarket wallets are of the [BIP32 Hierarchical Deterministic wallet](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki) form, starting from the root m/0 (here '0' marks the default account, in accordance with the BIP spec). The branches are defined by:

    m/0/mixdepth/[external/internal]

The value of mixdepth runs from 0..M-1, where M is the number of mixdepths chosen by the user; by default 5. The value of [external/internal] is 0 for external, 1 for internal. Thus a default wallet will contain 10 separate branches.

Note that all of the keys are of the non-hardened type.

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

Balances displayed by [`wallet-tool.py`](#the-wallet-toolpy-script) **include unconfirmed amounts** by default. This (i.e. treating unconfirmed coins as available) is **not currently supported** when running [bot entities](#entities), based on the problematic nature of spending unconfirmed coins, but can be done by editing configuration. For details on the fine-grained control over what conis are considered available, see the advice in the default version of `joinmarket.cfg` that is held in the file `configure.py`.


## Use of external and internal branches.

Payments into the wallet should be made into new addresses on the `external` branch for any mixdepth. For the above wallet, `muaApeqh9L4aQvR6Fn52oDiqz8jKKu9Rfz` (from mixdepth 2) or `mrgNjQWjrBDB821o1Q3qG6EmKJmEqgjtqY` (from mixdepth 4) would be suitable candidates. The index of the address on the branch is shown as the final 3 digit integer in the identifier. As usual with deterministic wallets, a configurable gap-limit variable is used to determine how far forwards to search after unused/new addresses are located.

In joinmarket [transactions](#transactions), a single destination output goes to the address designated by the transaction initiator (which need not be an address in a joinmarket wallet; it could be any valid Bitcoin address, including P2SH). The remaining outputs go to internal addresses as follows:
- If the transaction initiator has any change left ("sweep" transactions send a precise amount, without leaving change), it is sent to a new address in the internal branch of the same mixdepth as the initiator's inputs.
- Each liquidity provider sends a single change output to a new address in the same mixdepth as its inputs.
- Each liquidity provider sends a single output (with size identical to that of the destination output) to a new address in the next mixdepth, wrapping back to the first (that is, the mixdepth in BIP32 branch zero) upon reaching `max_mix_depth`.

The logic of this is fairly straightforward, and central to how Joinmarket works, so make sure to understand it: **the coinjoin outputs of a transaction must not be reused with any of the inputs to that same transaction, or any other output that can be connected with them, as this would allow fairly trivial linkage**. Merging such outputs is avoided by picking the inputs for a transaction only from a single mixdepth (although both internal and external branches can be used).

## `Wallet` object

The `Wallet` class is found in the module `joinmarket.wallet`. Its member variables, which are persisted to disk, are:

    addr_cache
    unspent
    seed
    gaplimit
    keys
    index

**addr_cache** is a dict, with each entry of format `Bitcoin address: (mixing depth, external/internal flag, index)`. The external/internal flag is 0/1 and the index is the index of the address on the branch. Note that the address itself **is not** persisted, only the index of the first unused key (and address) on each specific branch (see [here](#wallet-persistence)). 

**unspent** is a dict, with each entry of format `utxo: {'address': address, 'value': amount in satoshis}`, where `utxo` has format txid:n as usual in Bitcoin wallets. This is the fundamental data structure that Joinmarket uses to decide which coins to spend in joins.

**seed** is the master secret of the BIP32 wallet. See [this section](#wallet-generation-and-access-control) for more details.

**gaplimit** is used to decide how many addresses to search forwards (through unused addresses on a branch) before giving up and assuming no more addresses on that branch have been used. This is as usual for HD wallets; the default value is 6.

**keys** is a list of pairs of parent keys that are used to generate the individual branches, i.e. it has the form: `[(key for mixdepth 0 external branch, key for mixdepth 0 internal branch), (key for mixdepth 1 external branch, key for mixdepth 1 internal branch), ...]`

TODO: add documentation on using imported keys.

**index** this (too generically named!) is a list of pairs of pointers into each of the branches, marking the first unused address in that branch, format `[[a,b],[c,d]...]` with each letter standing for a positive integer. Note that this **is** [persisted](#wallet-persistence) to file storage to prevent address reuse in case of failures.

## Wallet generation and access control

The master private key is generated via a call to Python's `os.urandom`, this being the interface to the underlying OS randomness source. It is then passed through two rounds of SHA256. It is a 32 byte random string (which is the advised seed length for BIP32).

This seed is used to generate the wallet structure described above and according to the BIP32 specification.

The recovery seedphrase is a 12 word phrase of the type used in earlier versions of Electrum, and using that code base, specifically see the functions `mn_encode` and `mn_decode` in the module `old_mnemonic.py`, which uses a 1626 word list, also found in that file (note: 1626<sup>12</sup> implies 128 bit security). A commonly asked question is whether and why not Joinmarket supports BIP39. For now, the answer to that question is that it isn't relevant; since JM's wallet uses this specific HD structure, designed to allow coinjoins to occur safely, it is not directly compatible with other wallets. This may change in the future, however.

The wallet seed is encrypted for persistent storage using AES in CBC mode, using the module `slowaes.py`. Note that a bug was found earlier in this module's handling of PKCS7, which could have allowed 'decryption' with a wrong password (to garbage), but this was [fixed](https://github.com/JoinMarket-Org/joinmarket/pull/191).

## Wallet persistence

The wallet is persisted to disk in this format:

    {"index_cache": [[113, 163], [149, 161], [154, 134], [128, 149], [135, 120]], "encrypted_seed":
    "15336f8220ee6da168c153e1e11c0402c862fa377d2422c27b5f92ed37d52a840d1a80c4eda7ca9731ec5e0ebfa92cdd",
    "creation_time": "2015/05/08 15:56:58", "network": "mainnet", "creator": "joinmarket project"}

**index_cache** is a list of 5 x 2 entries, one for each branch in the wallet (in the default case) as described above. Each number is a pointer to the next unused address on the branch. This is of high importance because it allows the entity in control of the wallet to ensure that an address on that branch is not reused in more than one transaction. To achieve this, it's necessary that the function `Wallet.update_index_cache` is called *immediately* a new transaction is *proposed* for that entity, even if the transaction fails to complete. Note that this can lead to practical difficulties (such as large wallet gaps) in the case of Sybil attacks or errors where a long string of proposed but incompleted transactions occurs.

**encrypted_seed** is a 48 byte hex encoded string - this is 3 AES blocks, the plaintext being the 32 byte seed, the final block's plaintext being entirely pkcs7 padding.

**creation_time**, **network** and **creator** are self-explanatory.

Wallet persistence to disk only occurs in these scenarios: when using [`wallet-tool.py`](#the-wallet-toolpy-script), either generating a wallet or importing keys, and in the initialisation of the `CoinJoinOrder` object, which calls `update_cache_index()` method of the `Wallet` object (to prevent address reuse).

However, it's important to remember that the Bitcoin blockchain itself is another deeper layer of persistence; on starting a `Maker` or `Taker` bot, the wallet's `unspent` variable is updated by querying the `sync_unspent` method of the `BlockchainInterface` instance (see [Blockchain Interface](#blockchain-interface)).

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

---

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

### Publishing orders

A Maker is by definition an entity that broadcasts its availability to do coinjoins. It published what are called [orders](#orders) - but note that functionally these are *offers*.

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

Note that it is also possible to manually specify makers with which the taker does not want to join, by amending the `ignored_makers` list. This is not currently part of the configuration and is not expected to be needed, usually.

### Taker implementations

For each of these, please see the user instructions specified in the output of `python scriptname.py --help`.

#### The sendpayment script

This is designed to implement the simplest scenario for a `Taker` : it starts, does one coinjoin, then quits. The expected functionality is as follows.

For the sequence of events, see the bullet point list in [Taker](#taker), bearing in mind that this is the only-one-coinjoin option.

The syntax is:

    python sendpayment.py [options] [wallet file] [amount] [destaddr]

To carry out a *sweep* (see item 4 in [joinmarket transaction types](#joinmarket-transaction-types)), the field `amount` must be set to zero. For other transactions, the amount must be specified in satoshis.

The `destaddr` is the address which will receive the coinjoin amount (the amount specified in `amount`). Note that specifying a non-p2pkh address (i.e. a p2sh address, usually for multisig - addresses starting with '3' on the main bitcoin network) is highly inadvisable, as it allows immediate linkage of that coinjoin output to the input and change address.

The default mixdepth from which the spend will occur is mixdepth 0. Otherwise, the user must specify the mixdepth with the `-m` flag.

The user also specifies the number of counterparties with which to join. Choosing -N 1 is not recommended since this allows the counterparty to know the taker's destination address. The default is 2, but numbers from 3-5 are probably most suitable. Larger figures suffer from larger transaction fees, as well as potential unreliability problems.

Once the configuration has been set, the sendpayment script starts a `PaymentThread` thread object, which does the following:

First, it extracts the list of available orders from its internal `self.db` object. Then, it estimates the bitcoin transaction fee (see the separate [section](#bitcoin-transaction-fees)).

Next, the most important part of the code is the method of choosing which orders, read from the orderbook, are to be used for the construction of the join transaction. See the functions `sendpayment_choose_orders` in `sendpayment.py` and `choose_sweep_orders` in `support.py`. Within these functions, the ranking of orders is performed by one of three functions found in `support.py`:

* pick_order
* cheapest_order_choose
* weighted_order_choose

with the last of the three being the default. The logic behind the default is deferred to a separate [section](#weighted-orders-choose). Also note that `pick_order` cannot be used in conjunction with sweeping.

Once the set of orders has been chosen and the fees set (and the coinjoin amount, for sweeps), the process continues as described above for Takers, i.e. `Taker.start_cj` is called with the defined parameters. The remaining execution is handled by the `CoinJoinTX` object.

**Pseudocode**

    function do_sendpayment(sweep):
     load wallet
     start msgchan
     read orders from pit
     set transaction fee estimate(sweep)
     choose orders using chosen algo (sweep)
     send !fill message to chosen counterparties and do encryption handshake
     receive utxos
     construct template transaction and send to all counterparties
     receive signatures
     broadcast transaction to bitcoin network
     quit

The parameter 'sweep' is used here to illustrate the fact that two parts of the processing are different dependent on whether the transaction is of the sweep type (*sweepjmtx*) or not - in which case it is the canonical type (*cjmtx*).

**Different logic for sweeps**: Sweep transactions have a small but technically very significant additional complexity: the transaction *amount*, i.e. the amount used for the coinjoin amount, is not known *until the list of orders is chosen*. This is because until the list of counterparty (maker) orders is chosen, the total coinjoin fee is not known (since each maker specifies their own coinjoin fee, and contribution to the total bitcoin transaction fee). The code handles this problem in the `choose_sweep_orders` function in `support.py`. It does this in an iterative process: first, the set of available orders is constructed. Then, each order is selected one by one based on the choose_orders algorithm the user preferred (which must be one of `weighted_order_choose` or `cheapest_order_choose`). Multiple orders from the same counterparty are rejected. Then, the coinjoin amount is calculated so as to leave zero change. Finally, each order in the chosen set is examined to see whether the newly calculated coinjoin amount is within its `minsize` and `maxsize` parameters. If not, that order is removed from the list, and the iteration continues until *all* chosen orders match with the calculated coinjoin size.

The extra logic for bitcoin transaction fee handling for sweeps is covered in the bitcoin transaction fee [section](#bitcoin-transaction-fees).

####The tumbler script

This is designed to allow a `Taker` to heavily (if not perfectly) delink the coins in the wallet by means of a long sequence of coinjoins. The expected functionality is as follows.

For the basic sequence of events, see the bullet point list in [Taker](#taker), bearing in mind that this is the multiple-coinjoin option.

The syntax is:

    python tumbler.py [options] [wallet file] [destaddr1] [[destaddr2] ...]

By default, the tumbler will follow the steps listed below. In this `cjmtx` and `sweepjmtx` are as mentioned above for sendpayment.

* User provides 1 or more addresses (the default is 3, although they can be added during the tumbler run) for payment along with several other configuration variables. See `--help` for this (long!) list of options.
* Starting from one mixdepth (by default 0), follow these steps for each mixdepth:
 * Run several cjmtx of varying amounts with some makers. Amount variation is according to a statistical distribution as specified in the options.
 * Run a final sweepjmtx spending all remaining utxos to the next mixdepth, or to a chosen external address if more than one external address is specified.
* Between each of these transactions is a randomised wait, again with a statistical distribution defined in the options.
* The final transaction sweep from the final mixdepth spends out to the (last) external address provided at the start, so is a sweepjmtx with a spend as described earlier.

**Pseudocode**:

    load wallet
    start msgchan
    generate list of tumbler transactions
    for each tx in list:
      read orders from pit
      read in tx variables: tx.N counterparties, tx.amount, tx.destaddr
      if tx.destaddr = 'ask':
        prompt user to provide address
      set transaction fee estimate
      choose orders using chosen algo
      send !fill message to chosen counterparties and do encryption handshake
      receive utxos
      construct template transaction and send to all counterparties
      receive signatures
      broadcast transaction to bitcoin network
      wait (lock) until 1 confirmation on network is seen
      wait tx.wait_time

Provision for failed makers is as specified in [handling unresponsive makers](#handling-unresponsive-makers), and provision is also made for the case where the orderbook has insufficient liquidity. The general philosophy is to not give up easily, as the code is usually intended to be running for a long time (a whole day is not uncommon).

---

# Messaging Layer

As mentioned in the section on [Entities](#entities), each participant in Joinmarket must connect to a messaging channel in order to communicate with other Joinmarket participants. The broad functionality, required for any messaging layer is (ordered by priority):

* Both broadcast and private messaging
* Low latency (sub-second at least)
* Availability (including DOS resistance)
* Anonymity or facility thereof using common technologies (specifically, Tor)
* Scalability
* Decentralization

Note that the list does *not* include encryption, because E2E encryption technology (including authentication and message integrity) can be used as a layer over the messaging channel. Thus while use of TLS for connection to the messaging layer may be desirable, it isn't necessary.

Scalability is important due to the use of broadcast messages. It would be desirable to minimise this requirement; for example, using federated servers which publish orders out-of-band.

Decentralization: not using message hubs/servers avoids trust issues and avoids potential censorship (can also help with availability, in some scenarios). Purely decentralized P2P messaging is a little difficult to achieve, especially for users who are not technically sophisticated (see: NAT punching).


The message channel's functionality is abstracted in the module `message_channel.py`. The message_channel class contains methods for *sending* messages and for *registering callbacks to receive messages*.

## Encrypted messaging

For the sake of privacy, it is required to end-to-end encrypt some part of the messages transferred between parties taking part in a coinjoin transaction. Note that this is not required for monetary security; that is handled by Bitcoin itself; technically it is also not *required* for coinjoin; a coinjoin in which the messages between participants are in plaintext over the wire would allow any passive observer to deduce the coin linkages, but not a future observer of the blockchain. Clearly, this distinction is somewhat academic and it would be highly undesirable to do coinjoins in this way entirely in plaintext over the wire.

The technology used for this purpose is Daniel J Bernstein's [NaCl](https://nacl.cr.yp.to/), which uses Curve25519 elliptic curve cryptography and the Salsa20 stream cipher, with Poly1305 as the MAC. The specific implementation of this is [libsodium](https://libsodium.org), with the Python binding [libnacl](https://libnacl.readthedocs.org/en/latest/).

This was chosen for its wide usage, high reputation in the community and a design based on a philosophy of "a highly secure default with as few options as possible". The particularly functionality used from the library is [authenticated encryption](https://libnacl.readthedocs.org/en/latest/topics/public.html) using keys derived via [ECDH](https://en.wikipedia.org/wiki/Elliptic_curve_Diffie%E2%80%93Hellman). Note that nonces for encryption are chosen randomly per-message **by default** in libnacl. This is worthy of note as nonce-reuse would be a critical security weakness.

For abstraction this functionality is exposed via a separate module in Joinmarket, `enc_wrapper.py` (this wrapper is currently entirely transparent).

## Messaging handshake.

**In the clear** :

    TAK: !fill <order id> <coinjoin amount> <taker encryption pubkey>
    MAK: !pubkey <maker encryption pubkey>

Both maker and taker construct a libnacl crypto `Box` object to allow authenticated encryption between the parties.
These Box objects are properties of the `CoinJoinTx` and `CoinJoinOrder` objects, so they are specific to 
transactions and not to `Maker` and `Taker` entities.

**Encrypted** :

    TAK: !auth <input utxo pubkey> <btc sig of taker encryption pubkey using input utxo pubkey>

(Maker verifies the btc sig; if not valid, connection is dropped - send REJECT message)

    MAK: !ioauth <utxo list> <coinjoin pubkey> <change address> <btc sig of maker encryption pubkey using coinjoin pubkey>

(Taker verifies the btc sig; if not valid, as for previous)

Because the `!auth` messages are under encryption, there is no privacy leak of bitcoin pubkeys or output addresses.

If both verifications pass, the remainder of the messages exchanged between the two parties will continue under encryption.

Specifically, these message types will be encrypted:

* `!auth`
* `!ioauth`
* `!tx`
* `!sig`

Note on the above: A key part of the authorisation process is the matching between the bitcoin pubkeys used in the coinjoin transaction and the encryption pubkeys used. This ensures that the messages we are sending are only
readable by the entity which is conducting the bitcoin transaction with us. To ensure this, the maker should not sign any transaction that doesn't use the previously identified input utxo as its input, and the taker should not push/sign any transaction that doesn't use the previously identified maker coinjoin pubkey/address as its output.

## Messaging protocol.

See separate [document](https://github.com/JoinMarket-Org/JoinMarket-Docs/blob/master/Joinmarket-messaging-protocol.md) TODO: Fix broken links.

---

# Blockchain Interface

The module `blockchaininterface.py` attempts to encapsulate all access to the Bitcoin blockchain, whether via a centralized API service or a local Bitcoin node. 

The abstract base class `BlockchainInterface` declares the following 6 abstract methods:

    @abc.abstractmethod
    def sync_addresses(self, wallet):
        """Finds which addresses have been used and sets
        wallet.index appropriately"""
        pass

    @abc.abstractmethod
    def sync_unspent(self, wallet):
        """Finds the unspent transaction outputs belonging to this wallet,
        sets wallet.unspent """
        pass

    @abc.abstractmethod
    def add_tx_notify(self, txd, unconfirmfun, confirmfun, notifyaddr):
        """Invokes unconfirmfun and confirmfun when tx is seen on the network"""
        pass

    @abc.abstractmethod
    def pushtx(self, txhex):
        """pushes tx to the network, returns txhash, or None if failed"""
        pass

    @abc.abstractmethod
    def query_utxo_set(self, txouts):
        """
        takes a utxo or a list of utxos
        returns None if they are spend or unconfirmed
        otherwise returns value in satoshis, address and output script
        """
        # address and output script contain the same information btw
    
    @abc.abstractmethod
    def estimate_fee_per_kb(self, N):
        '''Use the blockchain interface to 
        get an estimate of the transaction fee per kb
        required for inclusion in the next N blocks.
	'''

Current, and any future, classes inheriting from `BlockchainInterface` must therefore implement these methods, which are more or less the required functionality for running either a `Maker` or a `Taker` entity in Joinmarket.

The current implementations are:

* `BitcoinCoreInterface`
* `BlockrInterface`
* `RegtestBitcoinCoreInterface`

`BitcoinCoreInterface` is in some sense the most fundamental case: Joinmarket is really designed to be run with a Core node, as this is much better for privacy than using an SPV wallet or a web API interface. This point is emphasized in the [Joinmarket wiki](https://github.com/JoinMarket-Org/joinmarket/wiki/Running-JoinMarket-with-Bitcoin-Core-full-node#running-joinmarket-with-bitcoin-core-full-node).

`RegtestBitcoinCoreInterface` is for testing using the regtest feature of Bitcoin Core. `RegtestBitcoinCoreInterface` inherits from `BitcoinCoreInterface` and adds a couple of extra features related to testing - the ability to trigger a new block, and the ability to make payments of arbitrary amounts of testnet coins to balances as required. There is more information on setup in [this wiki page](https://github.com/JoinMarket-Org/joinmarket/wiki/Testing). This testing setup could be streamlined and improved, but the fundamental idea is important: it is possible to simulate an entire joinmarket trading pit using a local IRC daemon and a Bitcoin regtest daemon, without having to worry about use of real bitcoins.

`BlockrInterface` uses the API from [Blockr](https://btc.blockr.io) to access blockchain data. Despite the strong caveats mentioned above, this is used by those without the patience or technical know-how to use Core.

A detail worthy of mention is that Blockr's API does not expose the rpc `estimatefee` which is needed for the abstract method `BlockchainInterface.estimate_fee_per_kb`; for this, the API of [blockypher](http://dev.blockcypher.com) is currently used, but it could of course change.

It is of course likely and desirable that other implementations be developed, e.g. `BlocktrailInterface` relying on the [Blocktrail](https://blocktrail.com) API. One could even envisage ameliorating privacy problems by mixing access to several (although this doesn't ameliorate, for example, the performance issue with polling web APIs).

## The Notify Thread

Both `Maker` and `Taker` entities need, in general, to be able to respond to the two principal "events" that can occur for a transaction on the Bitcoin blockchain: (1) transaction acceptance into a local memory pool and (2) transaction getting mined into a block for the first time. Of course (1) is a rather nebulous "event" since it's not global and not necessarily final, but for practical purposes it's treated as an event. As was discussed in the [Maker](#maker) and [Taker](#taker) sections, these events trigger callbacks `unconfirmfun` and `confirmfun`, which vary per entity. Triggering this requires "listening" to the blockchain. 

Thus, after a `Maker` or `Taker` has completed transaction negotiation with their counterparty, they access the global bc_interface (by calling `configure.jm_single().bc_interface`) and start a `NotifyThread`. This is done via a call to `add_tx_notify` (TODO why isn't this an abstract method? don't all blockchaininterface instances need it?). 

The call to `add_tx_notify` appends an entry to a list named `txnotify_fun`, which is a list of tuples of the format `(tx_output_set, unconfirmfun, confirmfun)`.

**For BitcoinCoreInterface**: On the first call to `add_tx_notify`, the `NotifyThread` thread is started. This thread starts an http server daemon, listening on the (host,port) specified in the configuration under section "BLOCKCHAIN" and settings "notify_host", "notify_port". 

The http daemon is an instance of `BaseHTTPServer.HTTPServer` and is instantiated with a class derived from `BaseHTTPServer.BaseHTTPRequestHandler` named `NotifyRequestHeader`. This class receives `HEAD` requests, specifically:

    /walletnotify?
    /alertnotify?

This corresponds to configuration in BitcoinCore that allows it to make HTTP requests whenever a wallet 'event' occurs (the arrival of (unconfirm) or confirmation of a transaction which is connected to the wallet). Note that the addresses in the **Joinmarket** [wallet](#wallets) are added as watch-only to Bitcoin Core, meaning that Core does not know their private keys but keeps track of them in a separate account. The account is named as 

    joinmarket-XXXXXX

(XXXXXX = first 6 characters of the hex encoding of double-sha256 of the first address in the external branch of the first mixdepth of the Joinmarket wallet). Thus Bitcoin Core is set up to fire `walletnotify` when an event happens for the `Maker` or `Taker`'s wallet.

`alertnotify` is triggered by alerts in BitcoinCore, which happens very rarely. The information in the alert is passed on to Joinmarket and it is displayed on the terminal as well as in the logs.

 On subsequent calls, extra callbacks (`unconfirmfun` and `confirmfun`) are added to the `bc_interface.txnotify_fun` list, to be called for each blockchain event. Over long periods of operation therefore, a long list of such notify functions could accrue; but since this is just a list of function pointers, it isn't important.

On reception of a `/walletnotify?txid` request, the `NotifyRequestHeader.do_HEAD` method is called, which checks the content of the transaction specified by `txid` by calling `getrawtransaction`, and then searches the set of elements of `BitcoinCoreInterface.notify_fun` to see if any of them contain the outputs of the transaction in their first element `tx_output_set` as described above. If so, it is those functions `unconfirmfun` and `confirmfun` which are the other two elements of the tuple in that list entry, which are called (which is called depends on whether the transaction data got from `getrawtransaction` shows the number of confirmations as zero or not).

**For BlockrInterface** : 

In this case, each call to `add_tx_notify` spawns a separate `NotifyThread` object. Hardcoded (currently) timeouts are used to decide when to stop listening for transaction arrival and confirmation:

    unconfirm_timeout = 10 * 60  # seconds
    unconfirm_poll_period = 5
    confirm_timeout = 2 * 60 * 60
    confirm_poll_period = 5 * 60

This is unfortunately a necessary design; we must poll with API calls to the remote server to update the state. No doubt, there are more sophisticated and efficient designs that could be looked into.

The thread is initiated with the transaction data and the specific `unconfirmfun` and `confirmfun` function pointers to trigger once the events occur on the blockchain.

---

# Bitcoin Transaction Fees

The earlier versions of Joinmarket used a fixed transaction fee, with a 10000 satoshi default and the ability to customise it by the Taker, using the `-f` option to `sendpayment.py` and `tumbler.py`. This is clearly not ideal, since the effectiveness of a particular transaction fee in allowing quick confirmation is a function of the size of the transaction in bytes. It was regularly observed that transactions are created with large size (say, 2-4 kB) and which were not being confirmed quickly.

## Fee Estimation Calculation

First, the size of a transaction in bytes must be estimated. This is currently found in `bitcoin.main.estimate_tx_size`, which shows the formula in comments:

    '''Estimate transaction size.
    Assuming p2pkh:
    out: 8+1+3+2+20=34, in: 1+32+4+1+1+~73+1+1+33=147,
    ver:4,seq:4, +2 (len in,out)
    total ~= 34*len_out + 147*len_in + 10 (sig sizes vary slightly)
    '''

To expand: the 34 \* (number of outputs in the bitcoin transaction) + 147\* (number of inputs) + 10 is estimated as the size of the transaction in bytes. **Note** that this is not correct for p2sh transactions, but a separate more sophisticated calculation including the possibility of mixed p2pkh and p2sh utxos is not yet implemented.

The output of this calculation is passed to `wallet.estimate_tx_fee()`, which takes the number of bytes and multiplies it by the estimated fee per kB (see the list of abstract methods in [BlockchainInterface](#blockchain-interface)). The estimated fee per kB is dependent on the configuration variable set in the config section POLICY and the variable `tx_fees`, which (confusingly?) is the number of confirmations to target. Thus a relatively impatient Taker may set `tx_fees` to 1, to target confirmation in the next block, and a relatively patient taker may set it to 3 or more.

---

# Orders and the trading pit

The joinmarket 'pit' is where communication takes place in order to coordinate joins. Messages are **broadcast** to the pit to update the state of maker bots, and to request order information from Maker bots. See [orders](#orders) below.

Further coordination between proposed participants in a join happens using **private** (not broadcast messages) to individual [entities](#entities), and this process is described in the [section](#messaging-layer) on the messaging layer, and the sub-document on the messaging [protocol](https://github.com/joinmarket-org/joinmarket-docs/Joinmarket-messaging-protocol.md).

The Joinmarket pit broadcast layer is public information, and it can be viewed by anyone, for example it is currently available [here](https://joinmarket.io).

## Public entity identities

Entities can connect to the pit over Tor, either directly or via the IRC hidden service. If future, different messaging layers than IRC are implemented, it is strongly intended to preserve this possibility (either using Tor or an equivalently strong anonymity layer). However direct network connections are also supported.

Thus, entities are intended to be anonymous by default, but they can use persistent naming if they choose. The default naming of entities is using a simple randomised string, the code for which is currently in `joinmarket.irc.random_nick()`. Thus, by default, an entity's 'name' in the pit will change on every restart.

## Orders

Orders must be of a recognised type, currently one of:

* relorder
* absorder

The list of valid order types is accessed in `joinmarket.configure.jm_single().ordername_list`. Orders of a type not included in this list will be ignored by Taker entities.

Several parameters are to be included with the order, currently:

* `cjfee`: the amount, in percentage or satoshis, required for the Taker to pay the Maker.
* `txfee`: the contribution, in satoshis, that the Maker will contribute to the overall Bitcoin transaction fee. Note: this may be deprecated in future version.
* `minsize`: the minimum size of the coinjoin output, in satoshis, that the Maker will agree to.
* `maxsize`: the maximum size of the coinjoin output, in satoshis, that the Maker will agree to.

The exact syntax of the messages to broadcast orders is found in the [protocol doc](https://github.com/JoinMarket-org/joinmarket-docs/joinmarket-messaging-protocol.md).

---

# The Configuration File

Todo.




