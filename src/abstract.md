# Abstract

Randomness is a basic building block for all sorts of applications. The use case range from lotteries that fully rely on randomness over games that may have some random elements to probabilistic modeling, simulations and governance applications.

Blockchains are systems in which every node in a decentralized network can validate the state of a replicated database by independently executing transactions and comparing the results. In order to do that, all computations need to be determinstic. When the node's specific context is accessed, such as the system's clock or a local random number generator, nodes may come to different results and fall out of consensus.

The operations performed on blockchains get more sophisticated every year. While Bitcoin only allows token sends, the next geneartion brought turing complete computations on chain (Ethereum), creating complex financial products and the first governance applications. Today we are seeing this idea laveraged more and more as the price for execution drops significantly on Layer 2 solutions and independent blockchains in Cosmos<sup>1</sup>. The usage of WebAssembly brings a sandboxing technology to blockchains that was designed to run at near native speed on today's CPUs (CosmWasm, NEAR). Multiple projects are working on rich governance applications and games<sup>2</sup>.

In this document, we describe how Nois brings decentralised randomness on chain and distributes it to a multitude of other IBC enabled chains in a very secure, fast, decentralised, cost efficient, and developer-focused manner.

## Footnotes

<sup>1</sup> Cosmos referes to the ecosystem of independent blockchains that communicate via IBC, not the Cosmos Hub.

<sup>2</sup> TODO: provide examples
