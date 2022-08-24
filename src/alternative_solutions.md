# Alternative Solutions

Many alternative approaches to various parts of the solution have been discussed and discarded. Here are some of those.

## Verify on each app chain

Instead of verifying the beacon on one chain and then sending it one could also verify the beacon once per app chain, i.e. have a Terrand-like instance on Terra, Juno, Tgrade, …. The bot network would then need to submit the beacon to each chain. This would remove the need for a Nois chain and IBC relayers. It could also remove the time between publication and callback.

However, drand verification comsumes a lot of gas and doing that once per chain is potentially inefficient. When blockspace is limited, the beacon submission transaction might not get committed for a long time.

There are pros and cons on both sides. But when thinking about hundreds of connected app chains the deduplication of the verification feels right. Also with IBC queries upcoming, [our state becomes your state](https://twitter.com/hdevalence/status/1555256686641786882).

## Implement Nois as a smart contract

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
