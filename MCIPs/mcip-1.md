---
MCIP: 1
title: NFT on-chain metadata
author: Francesco Sullo <francesco@sullo.co>
type: Standards Track
status: Draft
created: 2021-11-11
requires: 165
---

## Simple Summary

A standard protocol for NFT on-chain metadata

## Abstract

The following standard allows for the implementation of a standard protocol for gaming NFTs and other NFTs that must be managed on pure decentralized application, i.e., at a smart contract level.

## Motivation

The standard ERC721 was born mostly for collectibles. However games on chain may need attributes and other information on chain to be able to play with it. For example, the [original EverDragons factory](https://github.com/mscherer82/everdragons/blob/master/everdragons-contract/contracts/EverDragonsFactory.sol) was implementing the following

```solidity
    struct Dragon {
        bytes32 name;
        uint24 attributes;
        uint32 experience;
        uint32 prestige;
        uint16 state;
        mapping(bytes32 => uint32) items;
    }
```

to manage mutable and immutable properties of the single dragon. The limit of this model is that the properties were predefined and a different game could not reuse the same approach.

We need a generic solution that establish basic rules and is flexible enough to manage most scenarios and make NFT really movable among games.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.


```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

/// @title IMCIP-1 On chain metadata
///  Note: the ERC-165 identifier for this interface is 0x4b291b8f.
/* is ERC165 */
interface IMCIP1 {
  /// @dev This emits when the attributes for a token id are set.
  event MetadataSet(uint256 _tokenId, Metadata _attributes);

  /// @dev This struct saves info about the token. Some of the fields
  /// will be defined as mutable, some as immutable.
  /// The contract must implement a bytes32 variable that defines the mutability
  struct Metadata {
    // version must be immutable
    uint8 version;
    // status is mutable, it should be managed with bitwise operators
    uint8 status;
    // It supports a maximum of 8 properties. Here a list of the initial
    // values starting for the first digit on the right:
    //   bit name           value in bits and bytes1
    //   bridged              1               1
    //   transferable         1 << 1          2
    //   burnable             1 << 2          4
    // with the same approach used, for example, in chmod.
    // For example, if a token at a certain moment, is burnable and transferable
    // the value will be (1 << 1) + (1 << 2) => 6
    // If it is bridged and, by consequence, not transferable and not burnable, except
    // than by the bridge, it should be 7
    // If a token is set as not transferable or not burnable, the ERC721 hook
    // _beforeTokenTransfer must be overwritten accordingly.
    // The first bit from the right — bridged — is important becomes there are so
    // many bridges who move tokens around and it is almost impossible to
    // know if a token is bridged or not, i.e., if it is available on the market.

    // list of attributes
    uint8[30] attributes;
    // Unnecessary fields can be set to zero.
    // If, for example, a field requires more than 256 possible value, two bytes can be used for it.
  }

  /// @notice Retrieve version, status and attributes
  /// @dev It returns a Metadata object
  /// @param _tokenId The id of the token for whom to query the attributes
  /// @return The attributes of the token
  function metadataOf(uint256 _tokenId) external view returns (Metadata memory);
}
```

## Rationale

The reason why ERC721 metadata are off-chain makes perfect sense in general, in particular for collectibles, but it does not allow pure on-chain games to interact with the NFT because they cannot access the metadata. This proposal adds a relatively inexpensive solution to it. The limit is that you can have at most 30 attributes. If an NFT has more than 30 different traits, the version can help because it can refer to different subgroups of attributes.

## Backwards Compatibility

This is totally compatible with the ERC721 standard.

## Implementations

[EverDragons2 MCIP1](https://github.com/ndujaLabs/everdragons2-core/blob/main/contracts/MCIP1.sol)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).