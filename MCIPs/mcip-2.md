---
MCIP: 2
title: NFT cross-chain contracts
author: Emanuele Cesena <emanuele.cesena@gmail.com>
type: Standards Track
status: Draft
created: 2021-11-15
requires: 721
---
Discussion at https://github.com/ndujaLabs/MCIPs/issues/5

## Simple Summary

A standard protocol to list the NFT contract addresses on multiple blockchains.

## Abstract

The following standard extends the [contract-level metadata](https://docs.opensea.io/docs/contract-level-metadata) to describe the behavior of the NFT smart contract across multiple blockchains, specifically to list the contract addresses on the different chains.

## Motivation

NFTs are a digital representation of an asset, wether it's a piece of art, a financial contract or an item within a game. As the capabilities of NFTs evolve, the concrete instantiation of a NFT on a specific blockchain looses importance, and what matters more and more is the NFT as an entity that can live across blockchains.

The ecosystem of cross-chain NFTs is still very fragmented. Although there exist marketplaces that support multiple blockchains and  bridges to move NFTs across different chains, there's still a lack of tools to build cross-chain NFTs and coordinate the behavior 
of a collection across chains.

This standard defines a minimal set of information that can be shared by a NFT project owner to list all addresses across blockchains, therefore informing external parties about how the collection is expected to behave. This list of addresses can be used by marketplaces to display a single coherent collection, or by bridges to know where to mint tokens when they move across chains.

## Specification

Contact URI for ERC721 or ERC1155, unchanged.
```
contract MyCollectible is ERC721 {
    function contractURI() public view returns (string memory) {
        return "https://metadata-url.com/my-metadata";
    }
}
```

Data returned from the URI:
```
{
  # unchanged
  "name": "...",
  "description": "...",
  "image": "...",
  "external_link": "...",
  "seller_fee_basis_points": ...,
  "fee_recipient": "...",

  # NEW
  # addresses of the NFT collections on different chains
  "chains": {
    "ethereum": "0x3b6aad76254a79a9e256c8aed9187dea505aad52",
    "polygon": "0x0ece...",
    "solana": "...",
    ...
  }

}

```

## Rationale

The NFT project owner needs to maintain control over the collection across all chains, for example they have to be able to validate ownership on multiple marketplaces.

This standard defines a minimal set of information to describe the NFT contract across blockchains.

Marketplaces can use this list to display cross-chain NFTs as a coherent collection. Validation is responsibility of the marketplace and can be further detailed in future standards, for example the owner could include digital signatures proving ownership on each chain.

NFT bridges can use this list to know where to mint a NFT when it moves across chains. Authorization to mint should be granted by the owner of the contract.

## Backwards Compatibility

Fully compatible with the ERC721 or ERC1155 standards.

## Implementations

TODO

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
