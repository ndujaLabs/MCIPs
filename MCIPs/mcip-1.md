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

/// @title MCIP-1 On chain metadata
///  Note: the ERC-165 identifier for this interface is 0x4b291b8f.
interface IMCIP1 /* is ERC165 */ {

  /// @dev This emits when the attributes for a token id are set.
  event AttributesSet(uint256 _tokenId, Attributes _attributes);

  /// @dev This struct saves info about the token. Some of the fields
  /// will be defined as mutable, some as immutable.
  /// The contract must implement a bytes32 variable that defines the mutability
  struct Attributes {
    // version must be immutable
    bytes1 version;

    // status is mutable, it should be managed with bitwise operators
    bytes1 status;
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
    bytes1[30] attributes;
    // Unnecessary fields can be set to zero.
    // If, for example, a field requires more than 256 possible value, two bytes can be used for it.
  }

  /// @notice Retrieve the index of the first mutable attribute
  /// @dev By convention, the attributes array first elements are immutable, followed
  /// by mutable attributes. To know which one are mutable, all we need is the index
  /// of the first mutable attribute
  /// @param _tokenId The id of the token for whom to query the first mutable attribute
  /// @return The index
  function firstMutable(uint256 _tokenId) external view returns (uint256);

  /// @notice Returns the index of the last supported attributes
  /// @dev An NFT can have a variable number of attributes or a fixed one.
  /// It returns the index of the last accepted attribute index. For example,
  /// if an NFT has 16 immutable traits, and 5 mutable one and no other can be added,
  /// it should return 16 + 5 - 1 => 20
  /// If there is no limit, if should return 29
  /// @param _tokenId The id of the token for whom to query the first mutable attribute
  /// @return The index
  function latestAttributeIndex(uint256 _tokenId) external view returns (uint256);

  /// @notice Retrieve the mutability of an attribute based on its index
  /// @dev It returns a boolean. Mutable: true, immutable: false. It should use
  /// firstMutable to get the value
  /// @param _tokenId The id of the token
  /// @param _attributeIndex The index of the attribute for whom to query the mutability
  /// @return The mutability
  function isMutable(uint256 _tokenId, uint256 _attributeIndex) external view returns (bool);

  /// @notice Retrieve version, status and attributes
  /// @dev It returns an Attributes object
  /// @param _tokenId The id of the token for whom to query the attributes
  /// @return The attributes of the token
  function attributesOf(uint256 _tokenId) external view returns (Attributes memory);

  /// @notice Sets the initial attributes of a token
  /// @dev Throws if the already set
  /// For example, an NFT can have a factory contract who manages minting and changes. In
  /// that case, only the factory contract should be allowed to execute the function.
  /// At first execution if should allow to set mutable and immutable attributes up.
  /// At further calls, it must revert if trying to change an immutable attribute.
  /// @param _tokenId The id of the token for whom to change the attributes
  /// @param _version The version. It must revert if a not supported version
  /// @param _initialStatus The initial value of the status.
  /// @param _initialAttributes The array of the initial attributes
  /// @return true if the change is successful
  function initUpdateAttributes(uint256 _tokenId, bytes1 _version, bytes1 _initialStatus, bytes1[] memory _initialAttributes) external view returns (bool);


  /// @notice Sets the attributes of a token after first set up
  /// @dev Throws if the sender is not an operator authorized in the contract.
  /// @param _tokenId The id of the token for whom to change the attributes
  /// @param _index The index of the attribute to be changed
  /// @param _value The value of the attribute to be changed
  /// @return true if the change is successful
  function updateAttribute(uint256 _tokenId, bytes1 _version, bytes1 _initialStatus, uint256 _index, bytes1 _value) external view returns (bool);

  /// @notice Changes the status
  /// @dev Throws if the sender is not an operator authorized in the contract. See above.
  /// @param _tokenId The id of the token for whom to change the attributes
  /// @param _shiftPosition The number of position to be left shifted
  /// For example, to change the transferability of token #12
  /// from 1 to 0, the operator contract should call
  ///    updateStatus(12, 1, 0);
  /// It should revert if the _shiftPosition is > 7
  /// @param _value The bool must be converted in 1 or 0 and left-shifted before being added
  /// or subtracted. In added, an OR can make it, if subtracted, something like
  /// return (255 & (~(1 << _shiftPosition))) & existingValue
  /// would make the job
  /// @return true if the change is successful
  function updateStatus(uint256 _tokenId, uint256 _shiftPosition, bool _value) external view returns (bool);

}
```

## Rationale

The reason why ERC721 metadata are off-chain makes perfect sense in general, in particular for collectibles, but it does not allow pure on-chain games to interact with the NFT because they cannot access the metadata. This proposal adds a relatively inexpensive solution to it. The limit is that you can have at most 30 attributes. If an NFT has more than 30 different traits, the version can help because it can refer to different subgroups of attributes.

## Backwards Compatibility

This is totally compatible with the ERC721 standard.

## Implementations

[EverDragons2](https://github.com/ndujaLabs/everdragons2-core) -- a practical implementation

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).