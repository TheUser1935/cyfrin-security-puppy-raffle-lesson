## Protocol Overview

**AIM:** This project is to enter a raffle to win a cute dog NFT

The protocol should do the following:

Call the enterRaffle function with the following parameters:
1. address[] participants: A list of addresses that enter. You can use this to enter yourself multiple times, or yourself and a group of your friends.
   1. @audit-question enter self multiple times but point 2 says no duplicate addresses?
2. Duplicate addresses are not allowed
   1. @audit-question See question placed against 1
   2. @audit-question is there reentrancy risk?
3. Users are allowed to get a refund of their ticket & value if they call the refund function
   1. @audit-question oint 5 highlights an address taking a cut of fee, is this accounted for in the refund?
4. Every X seconds, the raffle will be able to draw a winner and be minted a random puppy
   1. @audit-question randomness issue if not done correctly?
5. The owner of the protocol will set a feeAddress to take a cut of the value, and the rest of the funds will be sent to the winner of the puppy
   1. @audit-question check how this is set
   2. @audit-question check maths of fee

## Possible invariants from docs/code

- no address can be entered more than 1 time
- random winner
- random NFT minted
- feeAddress gets cut of the total pool, rest goes to winner

### Audit scope details:

commit hash: e30d199697bbc822b646d76533b66b7d529b8ef5

In scope:

```
./src/
└── PuppyRaffle.sol
```

Compatibilities

Solc Version: 0.7.6
Chain(s) to deploy contract to: Ethereum

@audit-question - version of solidity is older than ^0.8.0 meaning that safemaths must be used due to integeter overflow

### Roles

Owner - Deployer of the protocol, has the power to change the wallet address to which fees are sent through the `changeFeeAddress` function. 

Player - Participant of the raffle, has the power to enter the raffle with the `enterRaffle` function and refund value through `refund` function

@audit-question check access control of `changeFeeAddress` function

## Scope Metrics

       1 text file.
       1 unique file.
       0 files ignored.

github.com/AlDanial/cloc v 2.00  T=0.01 s (93.2 files/s, 20139.8 lines/s)
-------------------------------------------------------------------------------
Language                     files          blank        comment           code
-------------------------------------------------------------------------------
Solidity                         1             30             43            143
-------------------------------------------------------------------------------

# Findings

1. MEDIUM - DoS attack in `enterRaffle` function
   1. PoC is shown in test file
2. HIGH - Reentrancy attack in `refund` function
3. Weak Randomness in pickWinner function - using blockchain and ms.sender to generate random number
4. HIGH - Integer overflow in pick winner for fees calculation - uses uint64 and is solidity version < 0.8.0
   1. Can cause extremely incorrect value for total fees
   2. We will have ETH stuck in our contract because we don't use .balance of contract and instead use a variable that undergoes integer overflow
5. There is precision loss when converting from uint256 to uint64 in pick winner function for calculating the fees
   1. We will have ETH stuck in our contract because we don't use .balance of contract and instead use a variable that is set using precision loss in casting
   2. unsafe cast from uint256 to uint64 causes wrap around and therefore our number is incredibly incorrect
6. Mishandling of ETH
   1. WithdrawFees function uses require statement that the balance of contract is equal to the totalFees
   2. WithdrawFees function will not be able to withdraw ETH if the balance is not equal to the totalFees
   3. Attacker could use selfdestruct(address(puppyRaffle)) to force the sending of ETH to the contract address
   4. This will cause the breaking of require statement and will lock the ETh of fees from being able to be withdrawn because the function will revert
7. Low - Missing events, or emitting bad events
   1. events aren't emitted when state changes
   2. reson for low is that events are super important to the ecosystem and heaps of tools read events to update their system, and if an event can be manipulated to affect other systems - then it is atleast a low
8. Info - _isActivePlayer is never used
   1. no need, not a vulnerability
   2. increases gas cost
   3. clutters code when not used
9. Low - getActivePlayerIndex will return zero if player is index zero
   1.  However, the function is meant to return '0' if a user has not entered
   2.  Meaning that a user who has entered might not know if they have entered or not
10. Info - Magic numbers
    1.  Magic numbers refers to random numbers used in statements of code. You should have a descriptor of what the number actually means and this can be done by making them a constant or immutable variable
11. Info - withdrawFees function has no access control
    1.  Anybody can call the withdraw function and it will send ETH to the `feeAddress`
12. Info - Constructor input validation
    1.  check that the fee address is not zero which would result in locked ETH until msg.owner changed the fee address
