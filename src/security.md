# Security

Nois Network provides a unbiasable publicly verifiable source of randomness to
dapps. We assume the following threat model:
## Security of the drand network

 At the moment, the Nois Network relies on the
 drand network. Drand relies on honest majority assumption, i.e. more than 50%
 of the drand operators must be honest, and so far this assumption has held in
 practice thanks to the variety in terms of nodes: different jurisdictions, platforms, 
 OS, and deployments. Drand has stood up through time as it is being used by multiple projects,
 including the popular Filecoin blockchain for more than 2 years without a single [downtime](https://status.drand.love/).

## Security of the Nois Network

 This is a new Cosmos based chain and therefore
  there is a list of validators responsible for running the consensus. On Cosmos
  the validators are ranked by reputation (highest uptime, number of chains etc).
  This list provdes a reliable source of validators already. The consensus relies 
  on the supermajority assumption, i.e. 2/3 + 1 of total stake must be held by 
  honests validators.
  In the future we want to expand our validator set to be more permissionless but 
  also containing some independent organizations similar to the drand network.

Given these two assumptions, the security of the model is pretty straightforward:
* The Nois smart contract guarantees the correct verification of any beacons
  submitted to it.
* The Nois chain guarantees the correct logic execution of the Nois smart
  contract, i.e. it will reply to IBC queries with exactly the correct beacon,
  already validated

## Application Security

Similar to regular applications, handling randomness is not a trivial task, and
a large number of vulnerabilities have had origins in the way to handle
randomness. 
Dapps need to have a randomness which has not been seen or biased with until a
certain point in time. For example, take a lottery dapps, no one should know the
randomness before the time allocated, otherwise one can cheat easily by
submitting the right ticket. At the time the lottery finish, a dapp should ask
for the _next_ randomness being available. This is exactly the main API endpoint
Nois contracts are offering. This reduces the chance of using the randomness
source "insecurely".

We also offer a "getRandomness(round)" endpoint that is to manipulate with care,
only for dapps developers that exactly know what they way, as this can be useful
for a certain number of use cases still.
  


