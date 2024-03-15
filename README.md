# BIP0352 Light Client Specification (WIP)
This repo has the goal to define a spec for a BIP 0352 Silent Payment light client. 
Additionally this repo should be a place for developers to inform themselves on the particularities on developing a light client for Silent Payments. This is not a repo where the entire protocol specification will be rehashed, but rather a place where special nuances will be highlighted, especially those relevant for light client developers. You can find the [BIP](https://github.com/josibake/bips/blob/silent-payments-bip/bip-0352.mediawiki) here.

## About BIP 0352
First let's start with brief explanation of what BIP0352 is. The BIP defines a protocol where a receiver publishes a static address from which the sender derives a public key of the receiver. The generated public key will be unique for every transaction and it won't be possible for an outside observer to attribute a certain public key to the receivers static address. The receiver has to then scan the blockchain in order to find UTXOs for his public keys. The receiver does not know which public keys belong to him before actually scanning and reengineering the computations that the sender made. This process is computationally intensive. For that reason light clients need a way to find their UTXOs without having to do all the computation themsel

## Tweak Index
In the BIP (mainly the appendix) a special focus is given to light clients and how those could be constructed. One idea presented is computing the tweaks on an indexing server and then providing the tweaks to light clients. This is a way to reduce the computational burden on light clients. They won't have to compute the input_hash, the public key sum and have to do one ECC multiplication less. [BlindBit-Backend](https://github.com/setavenger/BlindBit-Backend/) is one implementation of such an indexing server. Below is a section on the data. I published the [data](#the-files-can-be-found-here), so other implementations can easily compare the tweaks, taproot UTXO set and filters. This also helps as a sanity check for my own implementation.

## Data
I indexed the entire mainnet from taproot activation (709632) up to 834761. Below is an overview of the db sizes right after syncing (`du -h -d 1`). The implementation used for the below data was [BlindBit-Backend](https://github.com/setavenger/BlindBit-Backend/) on commit [e97997e1](https://github.com/setavenger/BlindBit-Backend/commit/48bfd9117c51be3e2dc4804d0ef1c817ace1fcc9). 
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

### The files can be found here:
- [Tweaks (cut-through)](https://snb-public.fra1.cdn.digitaloceanspaces.com/BIP0352/Reference-Indices/blind-bit-2024-03-15/tweaks-1710486950.csv)
- [UTXO Set](https://snb-public.fra1.cdn.digitaloceanspaces.com/BIP0352/Reference-Indices/blind-bit-2024-03-15/utxos-1710486950.csv)
- [Filters](https://snb-public.fra1.cdn.digitaloceanspaces.com/BIP0352/Reference-Indices/blind-bit-2024-03-15/filters-1710486950.csv)
- [Tweak Indices](https://snb-public.fra1.cdn.digitaloceanspaces.com/BIP0352/Reference-Indices/blind-bit-2024-03-15/tweak-indices-1710486950.csv)
- [Headers](https://snb-public.fra1.cdn.digitaloceanspaces.com/BIP0352/Reference-Indices/blind-bit-2024-03-15/filters-1710486950.csv)
- [Tweak Count Comparison](https://snb-public.fra1.cdn.digitaloceanspaces.com/BIP0352/Reference-Indices/blind-bit-2024-03-15/tweak_counts_comparison-1710486950.csv)
