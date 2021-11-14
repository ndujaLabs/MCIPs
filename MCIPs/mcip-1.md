---
MCIP: 1
title: NFT on-chain metadata
author: Francesco Sullo <francesco@sullo.co>
type: Standards Track
status: Draft
created: 2021-11-11
requires: 165
---
Discussion at https://github.com/ndujaLabs/MCIPs/issues/3

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
///  Version: 0.0.2
///  Note: the ERC-165 identifier for this interface is 0x0e32e192.
interface IMCIP1 /* is ERC165 */ {

  /// @dev This emits when the metadata for a token id are set.
  event MetadataSet(uint256 _tokenId, Metadata _metadata);

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
    // the value will be  (1 << 1) | (1 << 2)  => 6
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
    // If, for example, a field requires more than 30 possible attributes, two bytes can be used for it.
  }

  /// @dev It returns the on-chain metadata of a specific token
  /// @param _tokenId The id of the token for whom to query the on-chain metadata
  /// @return The metadata of the token
  function metadataOf(uint256 _tokenId) external view returns (Metadata memory);

  /// @notice Retrieve the mutability of an attribute based on its index
  /// @dev It returns a boolean. Mutable: true, immutable: false.
  /// @param _tokenId The id of the token
  /// @param _attributeIndex The index of the attribute for whom to query the mutability
  /// @return The mutability
  function isAttributeMutable(uint256 _tokenId, uint8 _attributeIndex) external view returns (bool);

  /// @notice Sets the attributes of a token after first set up
  /// @dev Throws if the sender is not an operator authorized in the contract.
  /// Specifically, the sender must be a platform approved buy the NFT contract owner 
  /// like, for example, a compatible game. Also, the token owner must approve the
  /// operator to spend their tokens (same function used in ERC721 for approvals). 
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
  /// @dev Throws if the sender is not an operator authorized in the contract. See above.
  /// Same like above for the approval.
  /// @param _tokenId The id of the token for whom to change the attributes
  /// @param _shiftPosition The number of position to be left shifted
  /// For example, to change the transferability of token #12
  /// from 1 to 0, the operator contract should call
  ///    updateStatus(12, 1, 0);
  /// @param _newValue The bool must be converted in 1 or 0
  /// @return true if the change is successful
  function updateStatus(
    uint256 _tokenId,
    uint256 _shiftPosition,
    bool _newValue
  ) external returns (bool);
}
```

## Rationale

The reason why ERC721 metadata are off-chain makes perfect sense in general, in particular for collectibles, but it does not allow pure on-chain games to interact with the NFT because they cannot access the metadata. This proposal adds a relatively inexpensive solution to it. The limit is that you can have at most 30 attributes. If an NFT has more than 30 different traits, the version can help because it can refer to different subgroups of attributes.

## Backwards Compatibility

This is totally compatible with the ERC721 standard.

## Implementations

[EverDragons2 ERC721WithMCIP1](https://github.com/ndujaLabs/everdragons2-core/blob/main/contracts/ERC721WithMCIP1.sol)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721Enumerable.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "./IMCIP1.sol";

contract ERC721WithMCIP1 is IMCIP1, ERC721, ERC721Enumerable, Ownable {
  event PlatformApproved(address platform);

  mapping(uint256 => Metadata) internal _metadata;

  uint8 internal _firstMutable;
  uint8 internal _lastMutable;

  mapping(address => bool) internal _platforms;

  // for version 1 of the MCIP-1, it must be 2
  uint256 public constant MAX_STATUS_SHIFT_POSITION = 2;

  modifier onlyApprovedPlatform(uint256 _tokenId) {
    require(_platforms[_msgSender()], "not an approved platform");
    require(_exists(_tokenId), "operator query for nonexistent token");
    address owner = ERC721.ownerOf(_tokenId);
    require(getApproved(_tokenId) == _msgSender() || isApprovedForAll(owner, _msgSender()), "spender not approved");
    _;
  }

  constructor(string memory name, string memory symbol) ERC721(name, symbol) {}

  function _beforeTokenTransfer(
    address _from,
    address _to,
    uint256 _tokenId
  ) internal override(ERC721, ERC721Enumerable) {
    if (_exists(_tokenId)) {
      require(_metadata[_tokenId].status & (1 << 1) == 1 << 1, "token not transferable");
      require(_to != address(0) || _metadata[_tokenId].status & (1 << 2) == 1 << 2, "token not burnable");
    }
    // else minting a new token
    super._beforeTokenTransfer(_from, _to, _tokenId);
  }

  function supportsInterface(bytes4 interfaceId) public view override(ERC721, ERC721Enumerable) returns (bool) {
    return interfaceId == type(IMCIP1).interfaceId || super.supportsInterface(interfaceId);
  }

  // approve a new game to manage the NFT's mutable attributes
  function approvePlatform(address _platform) external onlyOwner {
    require(_platform != address(0), "address 0x0 not allowed");
    _platforms[_platform] = true;
    emit PlatformApproved(_platform);
  }

  // return the index of first mutable attribute
  function firstMutable() public view returns (uint8) {
    return _firstMutable;
  }

  // return the index of last mutable attribute
  function lastMutable() public view returns (uint8) {
    return _lastMutable;
  }

  function metadataOf(uint256 _tokenId) public view override returns (Metadata memory) {
    return _metadata[_tokenId];
  }

  // solhint-disable-next-line
  function isAttributeMutable(uint256 _tokenId, uint8 _attributeIndex) public view override returns (bool) {
    return _attributeIndex >= _firstMutable && _attributeIndex <= _lastMutable;
  }

  function _initMetadata(
    uint256 _tokenId,
    uint8 _version,
    uint8 _initialStatus,
    uint8[30] memory _initialAttributes
  ) internal returns (bool) {
    require(_version == 1, "version not supported");
    _metadata[_tokenId] = Metadata(_version, _initialStatus, _initialAttributes);
    return true;
  }

  function updateAttributes(
    uint256 _tokenId,
    uint8[] memory _indexes,
    uint8[] memory _values
  ) public override onlyApprovedPlatform(_tokenId) returns (bool) {
    require(_indexes.length == _values.length, "inconsistent lengths");
    for (uint256 i = 0; i < _indexes.length; i++) {
      require(isAttributeMutable(_tokenId, _indexes[i]), "immutable attributes can not be updated");
      _metadata[_tokenId].attributes[_indexes[i]] = _values[i];
    }
    return true;
  }

  function updateStatus(
    uint256 _tokenId,
    uint256 _shiftPosition,
    bool _newValue
  ) public override onlyApprovedPlatform(_tokenId) returns (bool) {
    require(_shiftPosition <= MAX_STATUS_SHIFT_POSITION, "status bit out of range");
    uint256 newValue;
    if (_newValue) {
      newValue = (1 << _shiftPosition) | _metadata[_tokenId].status;
    } else {
      newValue = (255 & (~(1 << _shiftPosition))) & _metadata[_tokenId].status;
    }
    _metadata[_tokenId].status = uint8(newValue);
    return true;
  }
}

```

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
