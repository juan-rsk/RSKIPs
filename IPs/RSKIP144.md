
|RSKIP          |144           |
| :------------ |:-------------|
|**Title**      |Parallel Transaction Execution for Unitrie |
|**Created**    |23-OCT-2019 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |3 |
|**Status**     | |

# **Abstract**

This RSKIP describes how miners partition transactions into disjoint sets and how full nodes should process transactions in order to be safely parallelized. 

# **Motivation**

RSK processes transactions from blocks one by one, in the specified order. This is because the final state after processing two transactions when applied in different order may differ. However most transactions do not use the same keys of the state and therefore they could be parallelized without interference.

There are several obstacles to parallelization. [RSKIP02] and [RSKIP04] explore different methods that worked prior the implementation of the Unitrie. 

This RSKIP proposes using a runtime method to partition the transaction set into threads similar to RSKIP04 but tailored for the Unitrie. Miners are forced to serialize transaction execution and at the same time discover runtime key-access overlaps. Once all transactions have been processed and the partition is created, an index is created holding the last transaction number for each thread (the "partition" field). Full nodes can use this index to split the transaction set and parallelize execution.

# Specification

The idea is that transactions in a block are divided into ``N`` partitions, where the first ``N-1`` partitions are executed in parallel and the last partition is executed after the others. We refer to the first ``N-1`` partitions as the parallel partitions and to the last partition as the sequential partition.

A new field ``partitionEnds`` is added to the block header. This field consists of an array of shorts that indicates at which position in the transaction list each partition ends. For example, in a block with 10 transactions, ``partitionEnds = [3, 6]`` indicates that the first partition contains transactions 0, 1 and 2; the second partition contains transactions 3, 4 and 5; and the last partition contains transactions 6 to 9. Values in ``partitionEnds`` must be greater than 0 and in ascending order. An empty ``partitionEnds`` indicates that all transactions go in the sequential partition, and the REMASC transaction is always the last transaction of the sequential partition. The maximum number of parallel partitions that the miner can specify is equal to the minimum number of cores required to run the RSK node.

During execution, when a storage key is read, it is marked with the thread index in a per-partition readMap. When a key is written, the key is marked in a per-partition writeMap map. When all transactions of a certain partition have been processed, the writeMap is scanned. When all parallel partitions have finished processing, the readMaps and writeMaps are "merged". This requires efficient maps that enable traversing the keys in ascending lexicographic order. If a key belongs to a writeMap and a readMap of another partition, then the block is considered invalid. If a key belongs to a writeMap and a writeMap of another partition, then the block is also considered invalid. Recursive deletes must be correctly and efficiently handled. Miners are incentivized to produce a valid and efficient partition because by doing so their blocks spread faster over the network.

When a miner executes transactions to create an (unsolved) block, the miner must execute the transactions serially and decide which partition the transaction belongs to. The block gas limit is replaced by a per-partition gas limit. We say that a transaction is *connected* to another transaction if:

1. Both transactions are from the same sender account
2. Both transactions write the same storage key
3. One transaction reads the same storage key that the other transaction writes

A transaction is connected to a partition if it is connected to any transaction in that partition. The miner maintains a writeMap and a readMap that store which partition/s write or read each storage key. The miner also maintains an accountMap that maps each sender account to a partition. The miner then executes transactions from the pool and keeps track of which storage keys the transaction reads and writes. After executing the transaction, the miner compares the transaction read and written keys with the readMap and writeMap. The following scenarios are possible:

1. The transaction is not connected to any existing partition:
    1. The miner assigns the transaction to an empty parallel partition
    2. If no empty partition exists, the miner assigns the transaction to the less full parallel partition
    3. If all parallel partitions are full, the miner assigns the transaction to the sequential partition 
2. The transaction is connected to one parallel partition:
    1. The miner assigns the transaction to the parallel partition
    2. If the parallel partition is full, the miner assigns the transaction to the sequential partition 
3. The transaction is connected to more than one parallel partitions:
    1. The miner assigns the transaction to the sequential partition
    2. Alternatively, the miner might merge the connected partitions and assign the transaction there

To prevent DoS attacks, miners only execute a transaction if it fits in the sequential set. In this way, miners are guaranteed to include the transaction in the block regardless of the read and written storage keys. After assigning a transaction to a partition, the miner updates the readMap and writeMap accordingly. When no more transactions can be included in the block, the miner executes the block in parallel to produce the final state.

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
