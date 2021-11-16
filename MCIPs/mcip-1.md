---
MCIP: 1
title: NFT on-chain metadata
author: Francesco Sullo <francesco@sullo.co>
version: 0.0.3
status: Draft
created: 2021-11-15
requires: ERC165
---

# Cross-game NFT metadata

Discussion at https://github.com/ndujaLabs/MCIPs/issues/3

## Simple Summary

A standard protocol for Cross-game on-chain NFT metadata

## Abstract

The following standard allows for the implementation of a standard protocol for gaming NFTs and other NFTs that must be managed on pure decentralized application, i.e., at a smart contract level.

## Motivation

The standard ERC721 works very well for collectibles, despite being introduced by a game. Games on chain may need attributes and other information on chain to be able to play with it. For example, the [original EverDragons factory](https://github.com/mscherer82/everdragons/blob/master/everdragons-contract/contracts/EverDragonsFactory.sol) was implementing the following

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
///  Version: 0.0.2
///  Note: the ERC-165 identifier for this interface is 0x079882ad.
interface IMCIP1 /* is ERC165 */{

  /// @dev Emitted when the metadata for a token id and a game is set.
  event MetadataSet(uint256 indexed _tokenId, address indexed _game, Metadata _metadata);

  /// @dev This struct saves info about the token.
  struct Metadata {
    // A game can change the way it manages the data, updating the version
    uint8 version;

    // status should be managed with bitwise operators
    // and it refers to the status of the token in a specific game
    uint8 status;
    // It supports a maximum of 8 properties managed with bitwise operators.
    // For now there are two proposed fields:
    //   name        value
    //   burnable    1                // for example a potion
    //   burned      1 << 1           // if burned on game A, it is still usable in game B
    // adding more is up the game.
    // If an NFT has been virtually burned, the property could be considered immutable.
    // However, in some other game, an NFT could be resuscitated (why not?)

    // list of attributes
    uint8[30] attributes;
    // Attributes can be immutable (for example because taken from the metadata.json)
    // or mutable, because they depends only on the game itself.
    // If a field requires more than 256 possible value, two bytes can be used for it.
  }

  /// @dev It returns the on-chain metadata of a specific token
  /// @param _tokenId The id of the token for whom to query the on-chain metadata
  /// @param _game The address of the game's contract
  /// @return The metadata of the token
  function metadataOf(uint256 _tokenId, address _game) external view returns (Metadata memory);

  /// @notice Initialize the attributes of a token
  /// @dev It must be called by a game's contract to initialize
  /// the metadata according to its own attributes.
  /// Question: is there any risk if an unwanted game set attributes?
  /// @param _tokenId The id of the token for whom to change the attributes
  /// @param _version The version of the metadata
  /// @param _initialStatus The initial status (if, for example, it is burnable)
  /// @param _values The actual attributes
  /// @return true if the initialization is successful
  function initMetadata(
    uint256 _tokenId,
    uint8 _version,
    uint8 _initialStatus,
    uint8[30] memory _values
  ) external returns (bool);

  /// @notice Sets the attributes of a token after first set up
  /// @dev It modifies attributes by id for a specific game. It must
  /// be called by the game's contract, after an NFT has been initialized.
  /// @param _tokenId The id of the token for whom to change the attributes
  /// @param _indexes The indexes of the attributes to be changed
  /// @param _values The values of the attributes to be changed
  /// @return true if the change is successful
  function updateAttributes(
    uint256 _tokenId,
    uint8[] memory _indexes,
    uint8[] memory _values
  ) external returns (bool);

  /// @notice Changes the status
  /// @dev Acts on the metadata related to the specific game's contract.
  /// @param _tokenId The id of the token for whom to change the attributes
  /// @param _position The position of the property starting from the right
  /// For example, to burn token #12 in a specific game
  /// the game contract should call:  updateStatus(12, 1, 0);
  /// @param _newValue The bool must be converted in 1 or 0
  /// @return true if the change is successful
  function updateStatusProperty(
    uint256 _tokenId,
    uint256 _position,
    bool _newValue
  ) external returns (bool);

}
```

## Rationale

The reason why ERC721 metadata are off-chain makes perfect sense in general, in particular for collectibles, but it does not allow pure on-chain games to interact with the NFT because they cannot access the metadata. This proposal adds a relatively inexpensive solution to it. The limit is that you can have at most 30 attributes managed by any approved game.

## Backwards Compatibility

This is totally compatible with the ERC721 standard.

## Implementations

[EverDragons2 ERC721CrossGameMCIP1](https://github.com/ndujaLabs/everdragons2-core/blob/main/contracts/ERC721CrossGameMCIP1.sol)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

// Author: Francesco Sullo <francesco@sullo.co>
// EverDragons2, https://everdragons2.com

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721Enumerable.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "./IMCIP1.sol";

contract ERC721CrossGameMCIP1 is IMCIP1, ERC721, ERC721Enumerable, Ownable {
  event PlatformApproved(address platform);

  mapping(uint256 => mapping(address => Metadata)) internal _metadata;

  constructor(string memory name, string memory symbol) ERC721(name, symbol) {}

  function _beforeTokenTransfer(
    address _from,
    address _to,
    uint256 _tokenId
  ) internal override(ERC721, ERC721Enumerable) {
    super._beforeTokenTransfer(_from, _to, _tokenId);
  }

  //  function getInterfaceId() external view returns (bytes4) {
  //    return type(IMCIP1).interfaceId;
  //  }

  function supportsInterface(bytes4 interfaceId) public view override(ERC721, ERC721Enumerable) returns (bool) {
    return interfaceId == type(IMCIP1).interfaceId || super.supportsInterface(interfaceId);
  }

  function metadataOf(uint256 _tokenId, address _game) public view override returns (Metadata memory) {
    return _metadata[_tokenId][_game];
  }

  function initMetadata(
    uint256 _tokenId,
    uint8 _version,
    uint8 _initialStatus,
    uint8[30] memory _initialAttributes
  ) external override returns (bool) {
    _metadata[_tokenId][_msgSender()] = Metadata(_version, _initialStatus, _initialAttributes);
    return true;
  }

  function updateAttributes(
    uint256 _tokenId,
    uint8[] memory _indexes,
    uint8[] memory _values
  ) public override returns (bool) {
    require(_indexes.length == _values.length, "inconsistent lengths");
    require(_metadata[_tokenId][_msgSender()].version > 0, "game not initialized");
    for (uint256 i = 0; i < _indexes.length; i++) {
      _metadata[_tokenId][_msgSender()].attributes[_indexes[i]] = _values[i];
    }
    return true;
  }

  function updateStatusProperty(
    uint256 _tokenId,
    uint256 _position,
    bool _newValue
  ) public override returns (bool) {
    require(_position < 8, "status bit out of range");
    require(_metadata[_tokenId][_msgSender()].version > 0, "game not initialized");
    uint256 newValue;
    if (_newValue) {
      newValue = (1 << _position) | _metadata[_tokenId][_msgSender()].status;
    } else {
      newValue = (255 & (~(1 << _position))) & _metadata[_tokenId][_msgSender()].status;
    }
    _metadata[_tokenId][_msgSender()].status = uint8(newValue);
    return true;
  }
}

```

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
