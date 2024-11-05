# BIP0352 Light Client Specification (WIP)

This repo has the goal to define a spec for a BIP-0352 Silent Payments (SP) light client.
Additionally, this repo should be a place for developers to inform themselves on the particularities on developing a
light client for Silent Payments. This is not a repo where the entire protocol specification will be rehashed, but
rather a place where special nuances will be highlighted, especially those relevant for light client developers. You can
find [BIP-352](https://github.com/josibake/bips/blob/silent-payments-bip/bip-0352.mediawiki) here. The current
specification is based on [BlindBit Oracle](https://github.com/setavenger/blindbit-oracle/) in combination
with [blindbitd](https://github.com/setavenger/blindbitd). I hope this will change through feedback from other
developers, and more implementations for light clients and indexing servers pop up. The goal here is to formalize a
standard for SP light clients and explore how other wallet backends like Electrum can implement support for an SP light
client. 
The ambition is to integrate serving Silent Payments indexing data into Electrum.

## About BIP 0352

First let's start with brief explanation of what BIP-0352 is. The BIP defines a protocol where a receiver publishes a
static address from which the sender derives a public key of the receiver. The generated public key will be unique for
every transaction, and it won't be possible for an outside observer to attribute a certain public key to the receivers
static address. The receiver has to then scan the blockchain in order to find UTXOs for his public keys. The receiver
does not know which public keys belong to him before actually scanning and reengineering the computations that the
sender made. This process is computationally intensive. For that reason light clients need a way to find their UTXOs
without having to do all the computation themselves.

## Reducing work

The goal of this specification is to minimize the computational and required bandwidth burden on light clients. This
should be achieved without compromising their privacy. In the BIP a light client is defined as a client which does not
have a connection to a personal electrum server or a bitcoin node. This opens up a question on how a light client finds
an output sent to it. The wallet needs to compute the tweaks and check whether an output exists in a block. This
section will outline a couple of components needed for a light client to be able to send and receive without access to
a full node or a personal electrum server.

In general the chosen approach for this specification is to not show interest in any particular UTXO or transaction to 
any indexing server. This means that only an interest in a block will be shown. The block data is simplified and 
condensed down to the essential information required to find and spend UTXOs.

### 1. Using a tweak index

In the BIP (mainly the appendix) a special focus is given to light clients and how those could be constructed. One idea
presented is computing the tweaks on an indexing server and then providing the tweaks to light clients. This is a way to
reduce the computational burden on light clients. They won't have to compute the input_hash, the public key sum and have
to do one ECC multiplication less. [BlindBit Oracle](https://github.com/setavenger/blindbit-oracle/) is one
implementation of such an indexing server.

The tweak index can be kept smaller if necessary. Implementing cut-through could reduce the number of tweaks that have to be served. If all taproot UTXOs of a transaction are spent we can remove the tweak for that transaction from the index. An initial analysis has shown that with current mainnet data as much as 38% of tweaks could be removed from the index.

While this method does save a lot of bandwidth it has some drawbacks for light clients. During a rescan light clients will not be able to find old transactions that have been affected by cut-through. 

### 2. Compact block filters

With BIP 158 filters a light client can check whether a UTXO exists in a block. After precomputing the possible outputs
based on the tweaks index a wallet can match those against a filter of a block. Checking whether an owned output
potentially exists in a block before the block data is retrieved reduces the networking overhead. To reduce the
bandwidth requirements even further [BlindBit Oracle](https://github.com/setavenger/blindbit-oracle/) uses taproot only
filters. As SP outputs are always taproot, clients only have to match against taproot outputs.

### 3. Reduced UTXO set

Once a client found a match on the filter it needs to find the corresponding UTXO(s) which were sent to it. BlindBit
Oracle provides the UTXO set to light clients on a per-block basis. The data for a specific UTXO contains everything
that the client needs to properly spend it later. Firstly, only taproot UTXOs are collected as SP outputs are always
taproot. Secondly, only the data necessary to identify and spend the UTXO are sent to the light client. Both of these
measures together can save a lot of bandwidth for a client.

## Workflow receiving

### Overview

Per block:

1. Fetch the tweaks (possibly filtered for dust limit)
2. Compute the possible pubKeys for n = 0 (this has to include the labels including negated labels)
3. Fetch taproot-only filter (BIP 158)
4. Compare the pubKeys against a taproot-only filter
    - If no match: go to 1. with block_height + 1
    - Else: continue with 5.
5. Fetch simplified UTXOs[^1]
6. Scan according to the [BIP](https://github.com/josibake/bips/blob/silent-payments-bip/bip-0352.mediawiki#scanning) (
   bonus points if you reuse the pubKeys from 2. instead of recomputing)
7. Collect all matched UTXOs and add to wallet
8. Go to 1. with block_height + 1
9. Extra: Keep an eye on used pubKeys[^2]

Steps 2+3 can be done in parallel. It's rather unlikely that a block has zero tweaks which is the only reason why one would not need to request the filter.

## Specification

The mandatory fields are the bare minimum for a client to function. It makes sense to include optional 
fields from the get go so clients don't have to get certain information in a second or third round of requests. 
Optional fields are based on what data is usually provided by BlindBit Oracle too smoothen some workflows.

### Tweaks

Tweaks MUST be provided as json arrays of compressed public keys (33byte) in hex format. (Should bandwidth critical
applications use byte serialisation for this? A json array of strings is easier for developers but will probably use
more bandwidth.). See example below:

```json
[
  "031799df7770cfdabe460e47f4571fe4ded7d7a7922afa0f5ad91753473269cdbb",
  "03b2bda3a513123fe810cb1e65cd8a71a2495ea9229c50e4acc8d4b5baa51b4147",
  "030acf723f1e3bfb392ccbe0460e28be1ac13cce512c3cfe618cce839aac6212bb",
  "03cdc8c9f07d9917ed05d20231c63cac058b54a4095f6cc2fc53fc876e1226b44b"
]
```

### Filters

There are two filters. A New UTXOs filter and a spent UTXOs filter. One to find received UTXOs and another to check 
whether a UTXO has been spent. The first filter just contains the x-only pubKeys of newly created taproot UTXOs for
a given block. The new UTXOs filter should only include those UTXOs that come from eligible transactions to cut down
on size. The spent UTXOs filter is based on shortended hashes of the spent outpoints salted with the block_hash 
(sha256(outpoint||block_hash)[:8]). Outpoints are (txid, least significant byte first || 4-byte vout, least 
significant byte first).

NOTE: The `filter_type` could be defined in a BIP but can also be omitted. Alternatively a custom field and enum
can be derived for the purpose of this specification, especially that this specification adds two new filter_types. 
`block_height` in general is a nice to have as it might make processing easier.

Mandatory:

- block_hash (required for the key in the GCS filter)
- data (depending on the requested filter)

Optional:

- filter_type
- block_height

```json
{
  "block_hash": "0000011c426633eb092a3c9c3b6d7c6857ee8ce116eed84b825ab175e8f1e946",
  "block_height": 194686,
  "data": "589b773311635db09fe14060b888cc466764377134255375b0e09138b027e880eb21ce3b100cd10c051fb7092b99f5aeb155257aa98c2e84628cd3b3bc5afc00000017f2a9c9e15b57320eca510b19dd1f245f41a6fb8e48167ad9845f5726f74d48f996cf985426c4fa4fff8ab112c5608000064029dc3782bbfc73000004f2a7713584ee7809b06b57026264066666b2d00000e362efbb3abacf39316e0c8ffe106215006bc3dd0a8e7d083e8b55f350511c2645af368e9e94a5c0192bed4b8985bbcfe1cec95763d94f33892880000000000250f966e601e452d11f10d101748d9c5b76fb0d74e8",
  "filter_type": 4
}
```

### UTXOs

In order for light clients to be able to find and spend UTXOs accurately the following fields are defined.

Mandatory:

- txid
- vout
- value
- scriptPubKey (should we use x-only pubKeys instead?)
- spent (as long as a transaction has not fully spent all its taproot outputs we cannot remove them from the
  outputs_to_check. Technically this can be checked by the wallet software but it would be VERY helpful to always just
  provide this information)

Optional:

- timestamp
- block_height or block_hash 

```json
[
  {
    "txid": "355cce4314a2238f45801c81c15098f83668284156e80998ab38ce87ef7524cc",
    "vout": 1,
    "value": 1000000,
    "scriptpubkey": "51204128643f9891f245d0184043c19bfccdd38aa3c6e0c6b43444b676e148383c76",
    "block_height": 0,
    "block_hash": "00000120c9a85327d21a09e9a232eb1783f6de3e32cfeba02d2200e80cc017d4",
    "timestamp": 1715205985,
    "spent": false
  },
  {
    "txid": "355cce4314a2238f45801c81c15098f83668284156e80998ab38ce87ef7524cc",
    "vout": 2,
    "value": 1000000,
    "scriptpubkey": "51206a47348664f251683c74506925aed12b28d12e5d67ecadf9e5f9367b8109acc1",
    "block_height": 0,
    "block_hash": "00000120c9a85327d21a09e9a232eb1783f6de3e32cfeba02d2200e80cc017d4",
    "timestamp": 1715205985,
    "spent": false
  },
  {
    "txid": "355cce4314a2238f45801c81c15098f83668284156e80998ab38ce87ef7524cc",
    "vout": 3,
    "value": 1000000,
    "scriptpubkey": "512028f45a781567e05b0ef692efbc71db73024476ad28be293c1fc682f3c6762990",
    "block_height": 0,
    "block_hash": "00000120c9a85327d21a09e9a232eb1783f6de3e32cfeba02d2200e80cc017d4",
    "timestamp": 1715205985,
    "spent": false
  }
]
```

### Spent UTXOs

NOTE: This is not implemented in any BlindBit software yet but will be soon.

Spent UTXOs should return the block_hash and an array of (sha256(outpoint||block_hash)[:8]).

```json
{
    "block_hash": "00000120c9a85327d21a09e9a232eb1783f6de3e32cfeba02d2200e80cc017d4",
    "data": [
        "a3e456fa"
        "62aebc34"
        "983a4fe1"
        "e3456afb"
    ]
}
```

## Marking UTXOs as spent

As mentioned before checking the state of a UTXO should not be done by checking a specific scriptPubKey with a *third 
party* Electrum server. Showing interest for a specific UTXO leaks privacy and should not be done in privacy focused 
setting. A better alternative is to have a filter which indicates if a UTXO is spent or not. Construction of the filter 
is based on the hash of the outpoint salted with the block_hash (sha256(outpoint||block_hash)[:8]). Outpoints are 
(txid, least significant byte first || 4-byte vout, least significant byte first).

If matching against the filter is positive the spent UTXOs can be downloaded as well. The computed 8byte_hashes then 
need to be compared against the downloaded hashes. Based on that the outpoints can be marked as spent.

Marking a UTXO as spent but unconfirmed has to be done inside the wallet locally. Using Electrum to follow the 
transactions should not be done for the above reason.

A light client wallet should not run on different devices/instances. This will lead to states of UTXOs getting messed 
up if they are not confirmed. Using a third-party full node/Electrum server to check UTXO states is possible but will 
leak privacy. If several instances are a requirement, running a full node is the best path to preserve privacy.

## Backup from seed

- Old transactions might not be found
    - An indexer may prune a transaction where all taproot outputs have been spent. In that case the spent UTXOs will
      not be found on a rescan. Then the wallet software will not be able to reconstruct the transaction history without
      additional external help
- Will take a long time (maybe we can find a solution for that as well)
    - Rescanning the entire chain (or from an activation height) means that several thousand blocks have to be
      processed. This number will only increase over time. Backing up a wallet birth-height together with the seed can
      obviously help here.
    - Creating UTXO backup files could be an option as well. The drawback here is that they have to be periodically be created/updated.


## Out-of-band notifications

The BIP hints that receivers can be tipped off by the sender on where to find their UTXOs. In order to not leak privacy
a secure private communications channel has to be established. Currently, it is not clear how this can be achieved in 
a simple interoperable way. 


## Dust limits

This [nostr post](https://njump.me/nevent1qqs92428jlhkdtpa9334whfagxx8genc6l3ggcvua859mu2k0fldvgqzypn4hp87wh3pd2u5036r3mj3njnhw5mkmhc9mt0m5cnch5qju8tjs47scs3)
by waxwing unveiled an interesting fact about the taproot UTXO set. I was able to verify the numbers with the UTXO data
that was collected during indexing of the chain for the light client implementation. The numbers below are from the data
which is also provided at the end. We can see that ~85% of UTXOs have a value of 1,000 sats or less. Based on this it
makes sense to implement a dust limit for light clients. As long as it's clearly communicated to users that UTXOs below
value x are not found by normal scanning it should not be a problem. The UX would massively improve as scanning time has
a pretty linear relation to the size of the UTXO set. One could set the value to 1,000 maybe even higher. Clients can be
flexible with that. One could even have an indexing server implementation that allows clients to dynamically set the
dust
limit when fetching tweaks and UTXOs. Setting the number slightly above 1,000 sats or maybe set the value to 2,000 sats
would reduce the scanning time by 85%. This would be a massive UX improvement. Furthermore, low value UTXOs are bad UX
in itself as users will not be able to spend such UTXOs economically as shown very nicely by
this [tool](https://jlopp.github.io/unspendable-utxo-calculator). A p2tr UTXO costs 68 bytes to spend. At a fee rate of
only 15 sats/vByte the fees exceed the value of the UTXO. Considering that nobody wants fees to chew up a significant
part of their UTXO, it does not seem unreasonable to set the dust limit to 3,750 sats or even 5,000 sats. At that point
one could even reduce the UTXO set that needs scanning by 90%. It has to be mentioned that UTXOs below the dust limit
will only not be found by conventional/default scanning. A user could always fall back to scanning the full index (will
take **a lot** longer) or out-of-band notifications with the sender.

```plaintext
Distribution value (in sats) of the taproot UTXO set

count       38860604.00  // total number of UTXOs in the set
mean          155129.58
std         92404414.33
min                1.00
0%                 1.00
10%              330.00
20%              546.00
30%              546.00
40%              546.00
50%              546.00
60%              546.00
70%              546.00
80%              546.00
82%              600.00
85%             1000.00
90%             3750.00
max     410000000000.00
```

## Separating Scanning and Spending

This specification also aims to provide a protocol to interact with selfhosted scanning software. 
In order to improve the UX for users we can construct separate programs for scanning and spending. BIP 352 already considered 
this which is why we have a spending and a scanning key. This separation can achieve at least two things:  
- Added security - The spend key is not stored on an always-on device, risk of loosing funds can be reduced.  
- "Faster" perceived scan times - imagining an always-on scan program. This program can run as a background task on a server 
similar to an indexing server (think: BlindBit Oracle or in general an Electrum server). 

The process would look as follows:

1) The user sets up a wallet with scan and spend keys
2) She takes the spend public key and scan secret key and starts a scan program on a server or node-in-a-box device
3) When she wants to see the status of her wallet she queries her scan program for the UTXOs via a simple API request
4) Her spending wallet with both full keys now has the latest state of her UTXOs
5) She can spend just like with any other wallet

The scan program ideally runs continuously and is therefore always "synced" up to the chain tip. Hence querying the program 
should always return the most recent state of the wallet. For the user who opens her wallet the UTXO are therefore always at hand.
This eliminates long scan times for mobile devices as they don't have to do any scanning at all. Simple API requests are all
it takes. The experience is basically the same as with traditional wallets combined with electrum. Spending is just a couple requests 
away. The caveat here is that the entire point of Silent Payments is to improve privacy. So a user *must* trust the scan program to not 
leak privacy, hence the scan program should be run on ones *own* server/node-in-a-box. [BlindBit Scan](https://github.com/setavenger/blindbit-scan) 
is one of the first scanners for Silent Payments. Part of the BlindBit suite there is also a Proof-of-Concept mobile spending app called 
[BlindBit Spend](https://github.com/setavenger/blindbit-spend). Those two programs together follow the flow outlined above. 
BlindBit Scan does the continuous scanning and BlindBit Spend allows a user to spend her UTXOs. BlindBit Scan is ready for 
Umbrel but has not been submitted yet, it can run in combination with BlindBit Oracle (both are available in the same fork of [Umbrel-Apps here](https://github.com/setavenger/umbrel-apps))

### Endpoint specifications

The following endpoints are defined:

#### Height

A simple endpoint which returns the last scanned height.

```json
{"height": 861234}
```

#### Address

Returns the silent payment address which is being scanned for.

```json
{"address": "sp1..."}
```

#### UTXOs

The UTXOs belonging to the wallet are shared via this endpoint. 
Importing this UTXO data is sufficient to spend the coins belonging to the wallet.
For added security this data can be checked against a private electrum server as well. 
Although this data would normally originate from a highly trusted source in the first place. 
Hence additional checks are not critical.

```json
[
  {
    "txid": "66cf6460207e957ff77b1cad191050a8623d36671e94a46813b4bc10e6b35b6c",
    "vout": 0,
    "amount": 12000000,
    "priv_key_tweak": "6ffdbe4ab9c40a43edf31394ba8226475e610cd1a0d5da808248c1c9d6d79056",
    "pub_key": "bea89f2f17a7f438f4d5ab495d9a68a5d8ed3c7b5166f7427a6c39e6d9e3b062",
    "timestamp": 1721944866,
    "utxo_state": "spent",
    "label": null
  },
  {
    "txid": "66cf6460207e957ff77b1cad191050a8623d36671e94a46813b4bc10e6b35b6c",
    "vout": 1,
    "amount": 55990460,
    "priv_key_tweak": "ce2be4d8974a0852ac6c791c8d6332a1044d1b5224dc4da36522e9ff49150a07",
    "pub_key": "9326bdcdd477bf09d4fd3e39af62d9b3b0e0526c02d66b7ad4e0f80430cc1527",
    "timestamp": 1721944866,
    "utxo_state": "unspent",
    "label": {
      "pub_key": "02504188df0e7d4c1559e8d7e1d4c4c417086824ff37ddd98afbcc3a461430f1bd",
      "tweak": "cea33be68bbdeed59859bbc3b3afc8798564d3afc2690d68bb7223cb0d481dc0",
      "address": "tsp1qqt7u5h5n4cw8yctkednnnydytcuwmhz5xkdv0qtmscx90dwu06s5yq62ft33x5a2c605knje7u7c6fmfjvmjkq5xpchzr5xlqzguhwcyfc8gw326",
      "m": 0
    }
  }
]
```

[^1]: In order to properly spend a UTXO the light client needs txid, vout, scriptPubKey and the value. An index server
can easily provide this data to light clients.

[^2]: Let's craft a realistic scenario. Bob sends coins to Alice via the SP protocol, Alice receives the coins. The
coins are now on a p2tr pubKey which does have a bc1p... address which is also valid. Now Bob crafts a new transaction
and sends coins to Alice but not the proper way through the SP protocol but directly to the bc1p... address of which he
knows that it belongs to Alice. When Alice scans the chain she will not find this Output. The tweak she will compute for
this new transaction will be different from before, and it will not match the old pubKey where she already received
funds. No SP implementation should ever allow this, yet this will most likely happen anyway due to user errors. For
this reason a wallet implementation should always track matched scriptPubKeys to find such UTXOs. Additionally, the
corresponding tweak for the private key should be kept as well in order to easily spend such UTXOs.
