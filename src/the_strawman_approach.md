# The Strawman Approach

Blockchain applications can access certain information from the environment, that are deterministic and appear to be random at first glance. But usually those values are predictable or can be manipulated. E.g.

1. Solidity's `block.timestamp` is set by a single miner.
2. Solidity's `blockhash` is a value the miner can influence.
3. The block time of Tendermint ([BFT Time]) is of low entropy and can be influenced.
4. Block height has very low entropy as the height in which a transaction is included can easily be guessed.
5. Another thing that was spotted in the wild is using signatures as randomness. A pre-defined signer is asked to sign a given challenge. However, [it turns out](https://medium.com/@simonwarta/signature-determinism-for-blockchain-developers-dbd84865a93e) that common signing algorithms produce a deterministic but not unique signature, such that the signer can choose whatever value suits them.

To mitigate the risk of misuse, the CosmWasm maintainers decided to [not expose block hash](https://github.com/CosmWasm/wasmvm/issues/133) as this may falsely be assumed to be unpredictable but can be influenced by the block proposer.

To get reliable randomness, we need to rely on better technology which may not
be embedded in the native blockchain.

[bft time]: https://docs.tendermint.com/master/spec/consensus/bft-time.html
