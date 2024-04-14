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