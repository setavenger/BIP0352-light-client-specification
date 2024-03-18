# BIP0352 Light Client Specification (WIP)
This repo has the goal to define a spec for a BIP 0352 Silent Payment (SP) light client. 
Additionally this repo should be a place for developers to inform themselves on the particularities on developing a light client for Silent Payments. This is not a repo where the entire protocol specification will be rehashed, but rather a place where special nuances will be highlighted, especially those relevant for light client developers. You can find the [BIP](https://github.com/josibake/bips/blob/silent-payments-bip/bip-0352.mediawiki) here.

## About BIP 0352
First let's start with brief explanation of what BIP0352 is. The BIP defines a protocol where a receiver publishes a static address from which the sender derives a public key of the receiver. The generated public key will be unique for every transaction and it won't be possible for an outside observer to attribute a certain public key to the receivers static address. The receiver has to then scan the blockchain in order to find UTXOs for his public keys. The receiver does not know which public keys belong to him before actually scanning and reengineering the computations that the sender made. This process is computationally intensive. For that reason light clients need a way to find their UTXOs without having to do all the computation themsel

## Tweak Index
In the BIP (mainly the appendix) a special focus is given to light clients and how those could be constructed. One idea presented is computing the tweaks on an indexing server and then providing the tweaks to light clients. This is a way to reduce the computational burden on light clients. They won't have to compute the input_hash, the public key sum and have to do one ECC multiplication less. [BlindBit-Backend](https://github.com/setavenger/BlindBit-Backend/) is one implementation of such an indexing server. Below is a section on the data. I published the [data](#the-files-can-be-found-here), so other implementations can easily compare the tweaks, taproot UTXO set and filters. This also helps as a sanity check for my own implementation.

## Data
I indexed the entire mainnet from taproot activation (709632) up to 834761. Below is an overview of the db sizes right after syncing (`du -h -d 1`). The implementation used for the below data was [BlindBit-Backend](https://github.com/setavenger/BlindBit-Backend/) on commit [48bfd911](https://github.com/setavenger/BlindBit-Backend/commit/48bfd9117c51be3e2dc4804d0ef1c817ace1fcc9). 
The filter size can potentially be reduced. The taproot-only filters are solely based on the scriptPubKeys of the UTXOs of a block at that time. There is no cut-through used for those yet. I expect that periodically recomputing old filters based on the current UTXO set will reduce the size of filters. All of these  optimisations are important as they can greatly reduce the workload for light clients.

Cut-through reduces the number of tweaks by ~38%. This can help light client because this basically means that they can reduce their computation time by ~38%. The db size does increase quite a bit. This is due to the key value schema used. For `tweaks` the key is blockHash [32] + txid [32] then the tweak [33] as value. The `tweak-index` database stores the tweaks concatenated and unordered per blockHash. So we have blockHash [32] and then varlength value which is [n * 33]. 

Something that I considered as well was replacing the blockHash with the blockHeight which would enable easy range look ups with leveldb and reduce storage size as well. 

```plaintext
217M	./filters
2.7G	./utxos
 16M	./headers-inv
 12M	./headers
2.8G	./tweaks        33,679,543 tweaks 
1.7G	./tweak-index   54,737,510 tweaks
7.4G	.
```

## Dust limits
This [nostr post](https://njump.me/nevent1qqs92428jlhkdtpa9334whfagxx8genc6l3ggcvua859mu2k0fldvgqzypn4hp87wh3pd2u5036r3mj3njnhw5mkmhc9mt0m5cnch5qju8tjs47scs3) by waxwing unveiled an interesting fact about the taproot UTXO set. I was able to verify the numbers with the UTXO data that was collected during indexing of the chain for the light client implementation. The numbers below are from the data which is also provided at the end. We can see that ~85% of UTXOs have a value of 1,000 sats or less. Based on this it makes sense to implement a dust limit for light clients. As long as it's clearly communicated to users that UTXOs below value x are not found by normal scanning it should not be a problem. The UX would massively improve as scanning time has a pretty linear relation to the size of the UTXO set. One could set the value to 1,000 maybe even higher. Clients can be flexible with that. One could even have a indexing server implementation that allows clients to dynamically set the dust limit when fetching tweaks and UTXOs. Setting the number slightly above 1,000 sats or maybe set the value to 2,000 sats would reduce the scanning time by 85%. This woudl be a massive UX improvement. Furthermore low value UTXOs are bad UX initself as users will not be able to spend such UTXOs economically as shown very nicely by this [tool](https://jlopp.github.io/unspendable-utxo-calculator). A p2tr UTXO costs 68 bytes to spend. At a fee rate of only 15 sats/vByte the fees exceed the value of the UTXO. Considering that nobody wants fees to chew up a significant part of their UTXO, it does not seem unreasonable to set the dust limit to 3,750 sats or even 5,000 sats. At that point one could even reduce the UTXO set that needs scanning by 90%. It has to be mentioned that UTXOs below the dust limit will only not be found by conventional/default scanning. A user could always fallback to scanning the full index (will take **a lot** longer) or out-of-bounds communication with the sender.

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

## Workflow receiving
### Overview
Per block:
1. Fetch the tweaks (possibly filtered for dust limit)
2. Compute the possible pubkeys for n = 0
3. Fetch taproot-only filter (BIP 158)
4. Compare the pubkeys (+ previous matched scriptPubKeys[^1]) against a taproot-only filter
    - If no match: go to 1. with block_height + 1
    - Else: continue with 5.
5. Fetch simplified UTXOs[^2]
6. Scan accodring to the [BIP](https://github.com/josibake/bips/blob/silent-payments-bip/bip-0352.mediawiki#scanning) (bonus points if you reuse the pubkeys from 2. instead of recomputing)
7. Collect all matched UTXOs and add to wallet
8. Go to 1. with block_height + 1


## Backup from seed

- Old transactions will not be found
- Will take a long time (maybe we can find a solution for that as well)
- ... WIP


### The files can be found here:
Blocks: 709632 -> 834761
- [Tweaks (cut-through)](https://snb-public.fra1.cdn.digitaloceanspaces.com/BIP0352/Reference-Indices/blind-bit-2024-03-15/tweaks-1710486950.csv)
- [UTXO Set](https://snb-public.fra1.cdn.digitaloceanspaces.com/BIP0352/Reference-Indices/blind-bit-2024-03-15/utxos-1710486950.csv)
- [Filters](https://snb-public.fra1.cdn.digitaloceanspaces.com/BIP0352/Reference-Indices/blind-bit-2024-03-15/filters-1710486950.csv)
- [Tweak Indices](https://snb-public.fra1.cdn.digitaloceanspaces.com/BIP0352/Reference-Indices/blind-bit-2024-03-15/tweak-indices-1710486950.csv)
- [Headers](https://snb-public.fra1.cdn.digitaloceanspaces.com/BIP0352/Reference-Indices/blind-bit-2024-03-15/filters-1710486950.csv)
- [Tweak Count Comparison](https://snb-public.fra1.cdn.digitaloceanspaces.com/BIP0352/Reference-Indices/blind-bit-2024-03-15/tweak_counts_comparison-1710486950.csv)

[^1]: Let's craft a realistic scenario. Bob sends coins to Alice via the SP protocol, Alice receives the coins. The coins are now on a p2tr pubkey which does have a bc1p... address which is also valid. Now Bob crafts a new transaction and sends coins to Alice but not the proper way through the SP protocol but directly to the bc1p... address of which he knows that it belongs to Alice. When Alice scans the chain she will not find this Output. The tweak she will compute for this new transaction will be different than before and it will not match the old pubkey where she already received funds. No SP implementation should ever allow this, yet this will most likely happen anyways due to user errors. For this reason a wallet implementation should always track matched scriptPubKeys to find such UTXOs. Additionally the corresponding tweak for the private key should be kept as well in order to easily spend such UTXOs.

[^2]: In order to properly spend a UTXO the light client needs txid, vout, scriptPubKey and the value. An index server can easily provide this data to light clients. 
