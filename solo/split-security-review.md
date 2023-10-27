# Split Auditing Report

## Severity

High Risk

## **Summary**

The Split Protocol assessment has revealed a total of 2 identified vulnerabilities, classified as follows:

High Severity Vulnerabilities:
1 unique high-severity issue was detected.

## Vulnerability Details

**High Risk Issues**

In the SplitVault.sol contract, within the depositNFT() function, a vulnerability exists in the require statement. This vulnerability arises from the assumption that the owner of the NFT can also be the approved spender for that NFT. This assumption is flawed because the getApproved() for an NFT will typically return a zero address when the owner is indeed the owner and not an approved spender. Furthermore, the isApprovedForAll () will return false in such cases, potentially leading to unintended behaviour.

https://github.com/0xfps/split/blob/9089d487b48f0f190dd5d7536973a4f1603afab3/src/contracts/SplitVault.sol#L40

## POC

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.8.20;

import {console, Test} from "forge-std/Test.sol";
import {Split} from "../src/contracts/Split.sol";
import {MyNFT} from "../src/contracts/mockup/SplitNft.sol";

contract TestSplit is Test {
    Split private split;
    MyNFT private nft;

    address private deployer;
    address private user;

    function setUp() external {
        // set deployer & user
        deployer = makeAddr("deployer");
        user = makeAddr("user");

        vm.startPrank(deployer);
        // deploy split
        split = new Split();

        // deploy nft contract and mint
        nft = new MyNFT();
        nft.mint(deployer);

        assertEq(nft.balanceOf(deployer), 1);

        vm.stopPrank();
    }

    function test_split() external {
        // Test for splitVault Address
        vm.startPrank(deployer);

        assertTrue(address(split.splitVault()) != address(0));
        split.fractionalize(nft, 1, "Sage", "SAG", 10000);

        vm.stopPrank();
    }
}

```

## Impact

This will lead to a DOS attack thereby making the contract unusable.

## Gas/QA Report

For state variables that are initialized within the constructor and remain constant throughout the contract's lifecycle, it is recommended to use the "immutable" mutability. This ensures that the values of these variables cannot be changed after deployment, enhancing security and reducing gas costs.

https://github.com/0xfps/split/blob/9089d487b48f0f190dd5d7536973a4f1603afab3/src/contracts/Split.sol#L17

Many of the functions and parameters in the code lack proper comments and documentation, which can make the code less understandable and maintainable.
