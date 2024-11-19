+++
start-date = "2024-10-26"
end-date = "2024-10-26"
total-time = "5h"
total-rewards = "113$"
author = "0xSolus"
+++

# Sherlock Audit Contest Details 
[Gamma Brevis Rewarder](https://audits.sherlock.xyz/contests/496?filter=results)  

# Findings Summary

| ID     | Title                                                                    | Severity | Status       | Reward |
| ------ | ------------------------------------------------------------------------ | -------- | ------------ | ------ |
| [H-01] | Users can only execute one claim for distribution.                       | High     | Valid        | 113$     |
| [H-02] | Rewards distributed are not tracked causing insolvency in the contract.  | High     | Invalid      | 0$     |
| [M-01] | Protocol Fee can be set up to get most of the createDistribution amount. | Medium   | Invalid      | 0$     | 
| [L-01] | Incorrect parameter for `MAX_DISTRIBUTION_BLOCKS`                        | Medium   | Invalid      | 0$    | 

# Detailed Findings

## [H-1] Users can only execute one claim for distribution.
If a user has 2 claims for different block ranges inside the same distribution. 
He will only be able to claim the first one, the second will always revert due to 
[GammaRewarder.sol#L172](https://github.com/sherlock-audit/2024-10-gamma-rewarder-noahfigueras/blob/main/GammaRewarder/contracts/GammaRewarder.sol#L172).

Assume there is a distribution created with a range from block 100-1000.
Let's say Bob has two valid claims for the same distribution:
1. Claim with range from block 200 - 400.
2. Claim with range from block 700 - 900.

In this scenario, Bob will only be able to claim his first claim as the second
one will always revert due to `claim.amount > 0` for the `distributionId`.

**Recommendation:**
Consider checking the block ranges to not overlap, instead of the claim amount.

## [H-2] Rewards distributed are not tracked causing insolvency in the contract. 
The `handleProofResult()` and `claimTest()` do not keep track of the rewards 
awarded by a `distributionId`. In a case where the rewards of the proofs allowed 
to claim, surpass the `realAmountToDistribute` for a specific `distributionId`, it 
will use tokens allocated for other distributions causing problems for other users 
to claim their rewards. 

Let's say a `distribution1` is created with `100 USDC`, with a `10% fee`, `1000 blocks`
starting at 0 and ending at 1000 and a `100 blocksPerEpoch`.

Let's say a `distribution2` is created with `100 USDC`, with a `10% fee`, `1000 blocks`
starting at 0 and ending at 1000 and a `100 blocksPerEpoch`.

```solidity
    uint256 fee = 100e6 * 1e8 / BASE_9; // -> 10 USDC
    uint256 realAmountToDistribute = _amount - fee; // 90 USDC
    uint256 amountPerEpoch = realAmountToDistribute / ((1000 - 0) / 100); // 9 USDC reward x epoch
```
There's a total of `180USDC` in the contract to be distributed `90USDC` to `distribution1`
and `90USDC` to `distribution2`. 

In the following scenario, it can happen that the total rewards of multiple users 
claiming rewards accounts to `95USDC` or that 1 user claims `distribution1` with 
`totalRewards = 95USDC`.

In this case `distribution1` awarded `5USDC` more than it was funded for. Causing
some users in the `distribution2` to not be able to claim their rewards due to 
lack of funds. 

Note also, that in the case that this is accounted with off-chain mechanism, 
`totalRewardAmount` which should be related to `distributionAmountPerEpochs` is 
also not checked and can differ.

NOTE: This could be fixed by sending some more `USDC` to the contract, but it 
is not an ideal solution, specially as anyone can create a distribution and the 
team will have to add that extra funds for the contract to be solvent. 

**Recommendation**
Track all the funds awarded by a `distributionId` on claims and ensure it doesn't 
get bigger than `realAmountToDistribute`. 

## [M-1] Protocol Fee can be set up to get most of the createDistribution amount.
The variable `protocolFee` determines the fees that are collected by the protocol
for the total amount of a distribution. This variable can be set to a maximum of
`1e9-1` by the owner, thus allowing the owner to steal most of the amount of a 
distribution. 

NOTE: Even though this is a permissioned variable, if the owner account gets 
hacked, this will allow the hacker to steal most of the tokens of all the new
distributions that get created.

Take a look at the following:
```solidity
function test_protocolFee() public {
    uint256 amount = 100_000e6;
    uint256 protocolFee = 1e9 - 1;

    uint256 fee = amount * protocolFee / BASE_9;
    console.log(fee, amount);
}

```
```
[PASS] test_protocolFee() (gas: 3309)
Logs:
  99999999900 100000000000

  Traces:
    [3309] ContractBTest::test_protocolFee()
        ├─ [0] console::log(99999999900 [9.999e10], 100000000000 [1e11]) [staticcall]
            │   └─ ← [Stop] 
                └─ ← [Return] 
```
**Recommendation:** 
Restrict the variable to a maximum reasonable amount for a fee. EX: 1e8 == 10%.

## [L-1] Incorrect parameter for `MAX_DISTRIBUTION_BLOCKS`
In optimism there's a block every 2sec that means that in 4 weeks there are 1_209_600 blocks 
and not `9_676_800` as stated.

**Recommendation:**
```solidity
uint256 public MAX_DISTRIBUTION_BLOCKS = 1_209_600; // Blocks for 4 weeks 
```

