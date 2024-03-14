# BIP0352 Light Client Specification (WIP)
This repo has the goal to define a spec for a BIP 0352 Silent Payment light client. 
Additionally this repo should be a place for developers to inform themselves on the particularities on developing a light client for Silent Payments. This is not a repo where the entire protocol specification will be rehashed, but rather a place where special nuances will be highlighted, especially those relevant for light client developers. You can find the [BIP](https://github.com/josibake/bips/blob/silent-payments-bip/bip-0352.mediawiki) here.

## About BIP 0352
First let's start with brief explanation of what BIP0352 is. The BIP defines a protocol where a receiver publishes a static address from which the sender derives a public key of the receiver. The generated public key will be unique for every transaction and it won't be possible for an outside observer to attribute a certain public key to the receivers static address. The receiver has to then scan the blockchain in order to find UTXOs for his public keys. The receiver does not know which public keys belong to him before actually scanning and reengineering the computations that the sender made. This process is computationally intensive. For that reason light clients need a way to find their UTXOs without having to do all the computation themsel

## Tweak Index
In the BIP (mainly the appendix) a special focus is given to light clients and how those could be constructed. One idea presented is computing the tweaks on an indexing server and then providing the tweaks to light clients. This is a way to reduce the computational burden on light clients. They won't have to compute the input_hash, the public key sum and have to do one ECC multiplication less. [BlindBit-Backend](https://github.com/setavenger/BlindBit-Backend/) is one implementation of such an indexing server. Below is a section on the data. I published the [data](#the-files-can-be-found-here), so other implementations can easily compare the tweaks, taproot UTXO set and filters. This also helps as a sanity check for my own implementation.

## Data
I indexed the entire mainnet from taproot activation (709632) up to 834636. Below is an overview of the db sizes right after syncing (`du -h -d 1`). The implementation used for the below data was [BlindBit-Backend](https://github.com/setavenger/BlindBit-Backend/) on commit [e97997e1](https://github.com/setavenger/BlindBit-Backend/commit/e97997e15df0b419b9a3d0fe2324953a751b7633). ~It becomes clear how important cut-through is. We have reduced the tweak db size by a factor of 100. Moreover, it has to be noted that the actual tweak data is less by an even higher factor. The `tweaks` db stores every tweak as it's own key (blockhash [32] + txid [32]) with the corresponing tweak [33] as value. This means a lot of the data in the `tweaks` db is actually just keys and not tweaks. In the end one technically only has to serve tweak data to light clients. So a great reduction in bandwidth is possible here. For the `tweak-index` db we store the blockhash as key and the value are all tweaks concatenated. So most of the data in the index are actually tweaks contrary to the `tweaks` db. How much data exactly has to be served still has to be analysed. My assumption for now is that it will be a lot less than the 15MB in the `tweaks` db.~ **Bug in implementation. New values will be published once reindexed with fix.**

More over filter size can potentially be greatly reduced as well. The taproot-only filters are solely based on the scriptPubKeys of the UTXOs of a block at that time. There is no cut-through used for those yet. I expect that periodically recomputing old filters based on the current UTXO set will greatly reduce the size of filters. All of these  optimisations are important as they can greatly reduce the workload for light clients. And as an extra nugget they also improve db reads and overall storage sizes.

Something that I considered as well was replacing the blockHash with the blockHeight which would enable easy range look ups with leveldb and reduce storage size as well. 

```plaintext
214M	./filters
 18M	./utxos
 13M	./headers-inv
9.4M	./headers
 15M	./tweaks
1.7G	./tweak-index
2.0G	.
```

### The files can be found here:
- ~[Tweaks (cut-through)](https://snb-public.fra1.cdn.digitaloceanspaces.com/BIP0352/Reference-Indices/blind-bit-2024-03-14/tweaks-1710408770.csv)~
- ~[UTXO Set](https://snb-public.fra1.cdn.digitaloceanspaces.com/BIP0352/Reference-Indices/blind-bit-2024-03-14/utxos-1710408770.csv)~
- [Filters](https://snb-public.fra1.cdn.digitaloceanspaces.com/BIP0352/Reference-Indices/blind-bit-2024-03-14/filters-1710408770.csv)
- [Tweak Indices](https://snb-public.fra1.cdn.digitaloceanspaces.com/BIP0352/Reference-Indices/blind-bit-2024-03-14/tweak-indices-1710408770.csv)
