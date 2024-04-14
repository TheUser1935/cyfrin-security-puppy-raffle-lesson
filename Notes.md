# Cyfrin - Puppy Raffle - Security and Auditing Lesson Notes

## Static Analysis

### Slither

```bash
usage: slither target [flag]

target can be:
        - file.sol // a Solidity file
        - project_directory // a project directory. See https://github.com/crytic/crytic-compile/#crytic-compile for the supported platforms
        - 0x.. // a contract on mainnet
        - NETWORK:0x.. // a contract on a different network. Supported networks: mainet,optim,goerli,sepolia,tobalaba,bsc,testnet.bsc,arbi,testnet.arbi,poly

Can also print the results out in a variety of ays using the `--print` flag and then passing in a printer name (shown in slither --help)

$ slither . --print human-summary

....

$ slither . --print contract-summary



```

slither also has native configuration for Foundry and Hardhat and we can actually just pass a single period `.` as the target and slither is smrat enouhg to figure out the framework we are using.
```
slither .

```

### Aderyin

Aderyin is a Rust based static analysis tool.

After installing everything, you can just run the command `aderyin` and it will print out a report in a folder that it details.

## Cyfrin's - Smart Contract Explotes - Minimised

[Cyfrin's - Smart Contract Exploits - Minimised](https://github.com/Cyfrin/sc-exploits-minimized) is a repository containing minimised examples of common exploits in smart contracts - such as DoS, reeantrancy, etc.

It includes links to various sites where the can be tested, visualised, and even found in CTF challenges.

## Denial of Service - DoS

Denial of Service attacks can be fairly straight forward conceptually, but are quite powerful. There are a few DoS attack types that are used.


### Quick Attacker Mindset Questions for DoS Attack Vectors

- Is there a loop with a non-defined ending index or size? Think of the size as being infinite and ask `can infinite deny the operation of the protocol due to time or cost?`
- What are the upstream or downstream implications (if any) for an, imagined, infinite sized array?
- Is it possible that the smart contract could be trying to send a token to an address that can't recieve it?
- What logic problems arise when the protocl can't complete some neccessecary pieces of logic? 
  - Is there broader denial of service of the protocol?
- What external calls exist in the protocol? 3rd party, transfer of ETH
  - Is there a way for these external calls to fail? 
  - Can attacker malciously revert instantly on recieve or fallback to cause flow on effects?
  - Is there a specified gas limit that can be xploited in external calls from the protocol to cause a revert?
  - If yes, will it casuse the tope level function to revert entirely? 
  - How doe sthis affect the system?


### Attack vectors

#### Ethereum Block Gas Limit

Ethereum block sizes are limited by setting block gas fee limits. An Ethereum block has a target size of 15 million gas and a `maximum limit of 30 million gas`

#### Gas costings DoS attack
An attacker may find a contract desinged for a lottery, DeFi protocol or some sort of financial reward and blow up the size of an array that is looped over to deny any other people interacting with this array because of the costs associated with it. Potentially exceeding the block gas limit.

We can often see DoS attacks in cases where there is looping over arrays. This scenario normally raises some eyebrows at potential gas costs, and it is corerct that this can be more gas expensive - and in fact, this is what begins to feed into the issue.

Here is a snippet of a loop from the sc-exploits-minimized repository:

```solidity

address[] entrants;


//@note this function checks to see if the msg.sender already exists in the entrants array
function enter() public {
        // Check for duplicate entrants
        for (uint256 i; i < entrants.length; i++) {
            if (entrants[i] == msg.sender) {
                revert("You've already entered!");
            }
        }
        entrants.push(msg.sender);
    }
```

In this function, we will loop through the address array `entrants` to see if the msg.sender address already exists in the array.

As the size of this array continues to grow in size, the computation cost - as well as gas costs - begin to rise. 

An easy way to think about this is if the array had 1000 addresses, to verify the msg.sender address doesn't exist in the array from 0 to 999, it would take way more gas before being pushed into the array. However, the 3rd address only has to loop 3 times and therefore the gas cost is way lower. Now imagine there is 10,000 addresses in the array!

We have created a situation where there is a possibility that this function of the protocol becomes unusable because of the gas cost to complete the checks! Essentially denying the service of the contract.

#### Denial of Smart Contract Logic DoS attack by an attacker not having expected logic to handle certain transactions

We can see this attack begin to arise when we are dealing with a protocol that attempts to carry out straigth forward logic processing but have not accounted for the contract address that is interacting with it to be misconfigured in a way that can affect other core logic functions of the protocol.

For example, if the part of a lottery or auction system picks a winner, or handles highest new highest bidder and is holding on to tokens, and attempts to transfer a token/s to required address - such as winner, or former highest bidder but that address causes a revert. We may deny the service of the rest of the protocol because the spefific fucntions is required to be completed before it can execute normal operations.

### Proof of Concept

You can write tests, using a framework like Foundry, that interacts with functions that loop over arrays and for each transaction we can capture the gas costs for each transaction and carry out an assertion that for each new item added to the array results in higher gas fees.

The sc-exploits-minimized repository contains some Foundry tests demonstrating this exact PoC.

### DoS Case Study with Owen Thurm of Guardian Audits

#### Case Study 1 - DoS Dividends from Bridges Exchage audit completed by Guardian Audits - possible to exceed block gas limit

https://github.com/GuardianAudits/Audits/tree/main/Bridges

The protocol used an unbounded array to maintain a list of addresses that would receive receive shares of dividend payments for holding certain tokens. The functionality of the protocol would allow an attacker to keep generating new addresses and purchasing/minting small amount of these dividend yielding tokens and keep on repeating this until the gas cost exceeded the block gas limit, preventing any dividends.

There were no real restrictions on an attacker being able to carry out this attack. The only check the protocl had in place was to check whether or not that address had already minted more than 0 tokens or not - and this can be easily engineered to bypass by making new addresses quickly. Plus, the protocl did not have any restrictions on amont to mint and therefore an attacker could cheaply mint a large number of tokens before the gas cost exceeded the block gas limit and denying any other dividends.

**Mitigiation**

Re-desing of how you manage holders and carry out these checks. Guardian audits, for this protocl, reccomended a couple of approaches:


>Process the users in smaller batches, set a cap on number of users who can receive dividends, or modify the dividend allocation logic entirely such that a for loop is not needed.
>
>For an alternative approach, see this “pointsPerShare” implementation:
https://github.com/indexed-finance/dividends/tree/master/contracts

#### Case Study 2 - DoS because addresses that can't accept certain tokens cannot be liquidated or ADL orders in GMX

In the GMX protocl, they has a boolean variable called `shouldUnwrapNativeToken` that is used in a transfer function to transfer native tokens out of the protocol to a receiver address if the token that the addresses position has is wrapped native token - e.g wrapped ETH.

The protocol essentially tries to get the locked ETH for the wrapped ETH and then transfers the native ETH to the receiver using `payable(receiver).call{ value: amount, gas: gasLimit }("")`.

If the address is a smart contract and has a standardised `receive()` or `fallback()` function, then there is no issues and the protocol acts as expected and important time sensitive things such as liquidations can occur. However, if the smart contract at the address cannot accept the native token or **the smart contract's `receive()` or `fallback()` functions force a high gas cost to complete (perhaps some arbitrary looping?) and exceed the gas limit set causing a revert**, then the protocol will not be able to liquidate or ADL orders, and we need to always have a way of liquidating. The result of this set up is **denying the service of the protocol because liquidations cannot occur and this impacts the safety of the protocol**.

**Mitigation**

Guardian Audits proposed a couple of mitigation strategies: 

1. First approach is setting the `shouldUnwrapNativeToken` variable to `false` so that the protocol transfers the ERC20 wrapped native token to the address, removing the potential of having a smart contract as a position holder that can't accept native token transfers and therefore reverting.
2. Attempt to complete the transfer but if transfer failed, don't revert yet, instead re-wrap thge token and then transfer the tokens to the address.