---
MCIP: 3
title: NFT stakeable without losing ownership
author: Francesco Sullo <francesco@sullo.co>
type: Standards Track
status: Draft
created: 2022-05-01
requires: 721
---

[comment]: <> (Discussion at https://github.com/ndujaLabs/MCIPs/issues/5)

## Simple Summary

A standard protocol to stake an NFT to a contract without losing ownership.

## Abstract

The following standard extends the ERC721 standard protocol to allow a staking pool to mark an NFT as staked, without requiring that the NFT is transferred to the pool.

## Motivation

NFTs are used for many scopes. When used for governance, the holder may have a problem deciding if keeping ownership of the token and voting power, or stake it in a pool to get rewards.

## Specification

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

```solidity
interface IERC721Stakeable /* is IERC165 */ {

  // specify if an NFT is currently staked
  function isStaked(uint256 tokenID) external view  returns (bool);

  // get the pool which is staking the NFT
  function getStaker(uint256 tokenID) external view  returns (address);

  // set a pool
  function setPool(address pool) external;

  // removes a pool
  function removePool(address pool) external;

  // checks if the owner has staked any of its NFTs
  // used by approval functions (which must be modified, see implementation)
  function hasStakes(address owner) external view  returns (bool);

  // marks an NFT as staked.
  // It must be called by an authorized pool
  function stake(uint256 tokenID) external;

  // unmarks a staked NFT
  // It must be called by an authorized pool
  function unstake(uint256 tokenID) external ;

  // this is an emergency call. If the pool does not exist anymore
  // allows to mark the asset as not-staked
  function unstakeIfRemovedPool(uint256 tokenID) external;
}
```

It requires that the approval functions in the NFT are modified. Here an example:

```solidity
  function approve(address to, uint256 tokenId) public override  {
    require(!isStaked(tokenId), "NFT is staked");
    super.approve(to, tokenId);
  }

  function getApproved(uint256 tokenId) public view override returns (address) {
    if (isStaked(tokenId)) {
      return address(0);
    }
    return super.getApproved(tokenId);
  }

  function setApprovalForAll(address operator, bool approved) override public {
    require(!approved || !hasStakes(_msgSender()), "At least one NFT is staked");
    super.setApprovalForAll(operator, approved);
  }

  function isApprovedForAll(address owner, address operator) public override view returns (bool) {
    if (hasStakes(owner)) {
      return false;
    }
    return super.isApprovedForAll(owner, operator);
  }
```


## Backwards Compatibility

Fully compatible with the ERC721 standards.

## Implementations

[Everdragons2 Genesis Token V2](https://github.com/ndujaLabs/everdragons2-core/blob/main/contracts/Everdragons2GenesisV2.sol) 

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
