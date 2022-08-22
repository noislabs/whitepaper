# Nois Network

Nois Network aims to bring randomness (or noise) to the Cosmos ecosystem by providing a safe and secure entropy source and distributing randomness in the form of [random beacons][cryptographic-beacons] to other Cosmos blockchains via [IBC].

## Abstract

Randomness is a basic building block for all sorts of applications. The use case range from lotteries that fully rely on randomness over games that may have some random elements to probabilistic modeling, simulations and governance applications.

Blockchains are systems in which every node in a decentralized network can validate the state of a replicated database by independently executing transactions and comparing the results. In order to do that, all computations need to be determinstic. When the node's specific context is accessed, such as the system's clock or a local random number generator, nodes may come to different results and fall out of consensus.

The operations performed on blockchains get more sophisticated every year. While Bitcoin only allows token sends, the next geneartion brought turing complete computations on chain (Ethereum), creating complex financial products and the first governance applications. Today we are seeing this idea laveraged more and more as the price for execution drops significantly on Layer 2 solutions and independent blockchains in Cosmos<sup>1</sup>. The usage of WebAssembly brings a sandboxing technology to blockchains that was designed to run at near native speed on today's CPUs (CosmWasm, NEAR). Multiple projects are working on rich governance applications and games<sup>2</sup>.

In this document, we describe how Nois brings decentralised randomness on chain and distributes it to a multitude of other IBC enabled chains in a very secure, fast, decentralised, cost efficient, and developer-focused manner.

## The Naive Approach

Blockchain applications can access certain information from the environment, that are deterministic and appear to be random at first glance. But usually those values are predictable or can be manipulated. E.g.

1. Solidity's `block.timestamp` can be set by the miner.
2. Solidity's `blockhash` is a value the miner can influence.
3. The block time of Tendermint ([BFT Time]) is of low entropy and can be influenced.
4. CosmWasm team decided to [not expose block hash](https://github.com/CosmWasm/wasmvm/issues/133) as this may falsely be assumed to be unpredictable but can be influenced by the block proposer.
5. Block height has very low entropy as the height in which a transaction is included can be guessed.
6. Another thing that was spotted in the wild is using signatures as randomness. A pre-defined signer is asked to sign a given challenge. However, [it turns out](https://medium.com/@simonwarta/signature-determinism-for-blockchain-developers-dbd84865a93e) that common signing algorithms produce a deterministic but not unique signature, such that the signer can choose whatever value suits them.

To get relyable randomness, we need to do better.

## Our Approach

Nois Network aims to provide a safe and secure solution to the problem that is native to the IBC world and provides the best possible user experience for a wide range of applications.

In contrast to other consensus algorithms, Tendermint-based blockchains do not need randomness at block production layer. As a consequence no random value can trivially be passed to applications from consensus. So our randomness solition will live entirely on the app level, allowing us to keep block production going even when there is a hickup in the the randomness part.

In the first iteration, Nois will use random beacons produced by the [drand] network, which is powered by a consortium of participants that generate randomness using multi-party computation. This is implemented using BLS threshold signatures, which produces unpredictable values that cannot be manipulated by any of the drand participants. The drand mainnet is instantiated by the [Legue of Entropy][loe], which has been operating it in production for more than two years. For example, Filecoin relies on drand for block production.

Drand random beacons can be submitted to blockchains that perform BLS signature verification. This way we can build a random oracle that securely brings randomness on chain. This method was [described and proven in 2020 for CosmWasm](https://medium.com/@simonwarta/when-your-blockchain-needs-to-roll-the-dice-ed9da121f590). A few months later, this proof of concept was turned into production by [Terrand](https://docs.terrand.dev/). BLS verification is a computationally heavy operation, but leveraging the strength of the Rust optimizer and Wasm's near native execution speed, Drand beacons could be verified for less than 3$ in gas fees on Terra.

The next step in the evolution is to make drand beacons easily accessible by as many dapps as possible in a way that is easy to use and affordable. In an ideal world a dapp developer would just do something like this:

```rust
// pseudo-code
let beacon: [u8; 32] = await getRandom();
```

We believe the burden of implementing drand verification once per contract or even once per blockchain is too much in an ecosystem that is preparing for thousands of independent and inter-connected blockchains. So instead of executing the drand verification on the chain of the dapp, IBC is used to request the beacon from the Nois Network.

### How it works

The following steps are taken to get the randomness:

1. Contract on a CosmWasm-enabled chain (such as Juno or Tgrade) sends a message to a Nois proxy contract on the same chain. A reply with further information regarding the job is sent to the original contract.
2. The proxy contract sends an IBC message to its couter-part on the Nois Network where the job is put in the queue.
3. Once the drand beacon of the correct round is released, a network of bots send it to the Nois Network for verification.
4. After successful verfication, the pending jobs for the round are processed. For every matching job, an IBC response with the beacon is sent.
5. The proxy contract receives the beacon and sends a callback to the original contract.

### Commit to Callback Time (C2C Time)

Each Drand round is published at fixed points in time calculated as follows:

```
publish_time := genesis + (round - 1) * 30
```

where `genesis` is the drand network's start time (UNIX timestamp), `round` is an incremening integer starting at 1 and `30` is the 30 seconds round time.

Once this `publish_time` is reached, the randomness needs to be considered public. No matter if the chain or the contract knows the value already, any user can know the random value by observing the off-chain drand network. So it is important that after `publish_time` no actions for that round are allowed anymore. Think of it as closing submission of lottery tickets. Now the round should be processed as fast as possible in order to reveal the results and allow to continue the operation in case the next steps depend on it.

The time between beacon publishing an the callback consists of the following components:

- **Bot submission:** Multiple bots should observe the drand gossip network through varius communication protocols. Once a new round is found, they should craft a transaction, sign it and sent it to the Nois mempool. The fastest submission of one of the bots matters. When well connected, this should be doable in 1 second.
- **Block inclusion time:** When the beacon is in the mempool, the chain should ensure it is included in a block as fast as possible. With 5 second block times and an inclusion in the first or second block, this should be up to 10 seconds. The same transaction processes pending jobs and sends IBC messages. The block is executed in well under 1 second.
- **IBC relaying:** An IBC relayer picks up the message from the Nois chain and relays it to the destination chain. This depends on well configured and well connected relayers as well as availability of block space on the destination chain. In order to be included as fast as possible, the relayer should pay a transaction fee that is accepted by all validators. But in a high trafic sitiation, we don't get guearantees to be included quickly. So 5-20 seconds is a reasonable estimate.
- **Acknowledgement:** An IBC acknowledgement is sent to Nois but this is nothing the contract needs to wait for.

Once those steps are done, the callback is executed within 30 seconds of publishing. On a well configured network, average timings can be much faster though.

#### Choice of round

The application needs to commit to a round number before the beacon is revealed.
Fortunately we have a relyable [BFT Time] but this is not perfectly accurate and can be behind. In case the contract thinks `publish_time` is not yet reached while the beacon is already published, an attacker can abuse the knowledge of the randomness. So we intoduce a duration `safety_margin` and require `publish_time` to be at least `safety_margin` after the current BFT time (`block_time`).

Using the formular from above, we want

```
(1) publish_time := genesis + (round - 1) * 30
(2) publish_time >= block_time + safety_margin

genesis + (round - 1) * 30 >= block_time + safety_margin
(round - 1) * 30 >= block_time + safety_margin - genesis
round - 1 >= (block_time + safety_margin - genesis) / 30
round >= ((block_time + safety_margin - genesis) / 30) + 1
```

This inequality is satisfied when doing

```
round := ceil((block_time + safety_margin - genesis) / 30) + 1
```

Assuming `safety_margin` is set generously to 2 seconds, the `round` calculated that way is 2-32 seconds in the future.

This calculation can be generalized if an end time should be set in advance instead of closing right away. With flexible end times, the publish times of beacons can be matched and only the safety margin need to be considered.

#### Summary

When comitting to a round in 2-32 seconds that needs roughly 10-30 seconds to be processed, you get a delay between commitment to callback of 12-62 seconds. Varous usecase specific optimization techniques are to be explored.

## Alternative Solutions

Many alternative approaches to various parts of the solution have been discussed and discarded. Here are some of those.

### Verify on each app chain

Instead of verifying the beacon on one chain and then sending it one could also verify the beacon once per app chain, i.e. have a Terrand-like instance on Terra, Juno, Tgrade, …. The bot network would then need to submit the beacon to each chain. This would remove the need for a Nois chain and IBC relayers. It could also remove the time between publication and callback.

However, drand verification comsumes a lot of gas and doing that once per chain is potentially inefficient. When blockspace is limited, the beacon submission transaction might not get committed for a long time.

There are pros and cons on both sides. But when thinking about hundreds of connected app chains the deduplication of the verification feels right. Also with IBC queries upcoming, [our state becomes your state](https://twitter.com/hdevalence/status/1555256686641786882).

### Implement Nois as a smart contract

A drand verifier that is accessible via IBC can be implemented on an existing chain with CosmWasm. This would be easier to start with and would not require a new token. However, going for a custom chain has the following motivation:

- Cosmos is an ecosystem of app specific chains where creating one chain is relatively easy.
  The mentality in the ecosystem is to have many chains that are independent and interconnected.
  The tooling and people are ready for many chains.
- In a world with competing app chains that host our users, running on one of those chains makes us biased towards this chain.
  Being neutral regarding chains is very helpful in politics.
- The randomness chain would not halt if the smart contract chains halts.
- The ability to create overlapping validator and drand MPC sets is a way to incentivise drand node operators and get new players into Cosmos.
- The following optimizations are possible:
  1. The drand verification contract is 550 KB large. Terrand had to split the code in two contracts to deploy it to Terra.
     A custom chain can allow lager contract sizes.
  2. The verification consumes significant block space and may get expensive on other general purpose chains.
  3. CosmWasm allows pinning contracts. Those contracts are kept in memory and are loaded and executed faster.
     We can utilize this feature on a custom chain.
  4. CosmWasm has a somewhat unknown cronjob feature that allows governance to run contract execution in every block.
     That’s useful to e.g. process queues.
  5. We can use chain governance to upgrade the contract which is more transparent and safer than multisig upgradability.
  6. Due to permissioned contract uploads the use of a Wasm compiler with unbound compile time becomes possible, which can lead to faster verification.

## Further Work

The solution above explains what we can do with technology available today. But we'll not stop there. The next steps might be the following (unsorted).

### High Frequency Beacons

Right now drand emits a new beacon every 30 seconds. But there are plans to reduce that round time to something like 3 or 5 seconds. This will allow Nois to operate on higher frequency.

### Short Block Times

The Nois network can consider to reduce block times from the typical 5-7 seconds in Cosmos to something shorter. Doing so has to be carefully tested in environments with many globally distributed validators. But seeing teams successfully testing 1s block times is promising.

### IBC Queries

We'll closely follow the development of IBC queries, which is a technology we assume to allow for faster round trips and lower fees.

### Alternative Entropy Sources

We love what drand and the League of Entropy brought to the internet, which is nothing less than the first decentralized entropy source. But choice is important. Some applications may wish to sacrifice some of the security properties in order to get faster and cheaper randomness. Some users might demand governance voting for drand [MPC] participants. The options are to be explored but the network will be designed for multiple sources.

### Internalizing randomness generation

The node operators of the Nois Network could integrate randomness generation into their own operations. There has been work to [integrate random beacons into Tendermint](https://medium.com/@dgaminghub/arcade-tendermint-hack-with-built-in-threshold-bls-random-beacon-for-applications-a51eafb77f53) that could be explored. But also a second process that runs along with the validator node would be an option.

## Definitions

In this context we talk about public randomness, i.e. values that are unpredictable for all participants but once they are revealed, they are free to share. Private randomness that needs to be kept secret by one or some participants is out of scope for this document.

## Footnotes

<sup>1</sup> Cosmos referes to the ecosystem of independent blockchains that communicate via IBC, not the Cosmos Hub.

<sup>2</sup> TODO: provide examples

<!-- links -->

[bft time]: https://docs.tendermint.com/master/spec/consensus/bft-time.html
[cryptographic-beacons]: http://www.copenhagen-interpretation.com/home/cryptography/cryptographic-beacons
[drand]: https://drand.love
[ibc]: https://ibc.cosmos.network/
[loe]: https://en.wikipedia.org/wiki/League_of_entropy
[mpc]: https://en.wikipedia.org/wiki/Secure_multi-party_computation
