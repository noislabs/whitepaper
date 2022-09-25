# Cost Efficiency

Nois uses highly efficient Tendermint, IBC and CosmWasm technology to provide a cost-efficient solution to users.

## Verify on each app chain

An alternative approach to verifying the beacon on one chain and then distributing it across chains would be to verify the beacon once per app chain, i.e. have a Terrand-like instance on Terra, Juno, Tgrade, …. The bot network would then need to submit the beacon to each chain. This would remove the need for a Nois chain and IBC relayers. It could also remove the time between publication and callback.

However, drand verification consumes a lot of gas and doing that once per chain is potentially inefficient. When blockspace is limited, the beacon submission transaction might not get committed for a long time.

There are pros and cons on both sides. When thinking about hundreds of connected app chains, the deduplication of the verification feels right. With IBC queries upcoming, [our state becomes your state](https://twitter.com/hdevalence/status/1555256686641786882) and thus it makes sense to have one "randomness" chain accessible from all chain very easily.

## The app chain model

A drand verifier that is accessible via IBC can be implemented on an existing chain with CosmWasm. This would be easier to start with and would not require a new token. However, going for a custom app chain has the following motivation:

- Cosmos is an ecosystem of app specific chains where creating one chain is relatively easy.
  The mentality in the ecosystem is to have many chains that are independent and interconnected.
  The tooling and people are ready for many chains.
- In a world with competing app chains that host our users, running on one of those chains makes us biased towards this chain.
  Being neutral regarding chains is very helpful in politics.
- The randomness chain would not halt if the smart contract chain halts.
- The ability to create overlapping validator and drand MPC sets is a way to incentivise drand node operators and get new players into Cosmos.
- The following optimizations are possible:
  1. The drand verification contract is 550 KB large. Terrand had to split the code in two contracts to deploy it to Terra.
     A custom chain can allow larger contract sizes and we now have the logic
     implemented in one contract.
  2. The verification consumes significant block space and may get expensive on other general purpose chains.
  3. CosmWasm allows pinning contracts. Those contracts are kept in memory and are loaded and executed faster.
     We can utilize this feature on a custom chain, and we can also reduce the
     cost of verifying the beacons.
  4. CosmWasm has a somewhat unknown cronjob feature that allows governance to run contract execution in every block.
     That’s useful to e.g. process queues.
  5. We can use chain governance to upgrade the contract, which is more transparent and safer than multisig upgradability.
  6. Due to permissioned contract uploads, the use of a Wasm runtime backend with a slower but stronger optimizer becomes possible,
     which can lead to faster verification.
