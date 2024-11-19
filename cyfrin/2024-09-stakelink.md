+++
start-date = "2024-10-7"
end-date = "2024-10-11"
total-time = "15h"
total-rewards = "442.16USDC"
author = "0xSolus"
+++

# Cyfrin Audit Contest Details 
[Liquid Staking - Stakelink](https://codehawks.cyfrin.io/c/2024-09-stakelink)  

# Findings Summary

| ID     | Title                                                                    | Severity | Status       | Reward      |
| ------ | ------------------------------------------------------------------------ | -------- | ------------ | ----------- |
| [H-01] | Missing Token Transfer During Withdrawal in OperatorStakingPool.         | High     | Valid        | 442.16 USDC |

# Detailed Findings


# [H-1] Missing Token Transfer During Withdrawal in OperatorStakingPool

## Summary

There's a critical issue in the `_withdraw()` function of the `OperatorStakingPool.sol` contract, where the withdrawal process does not actually transfer tokens back to operators. This leaves operators unable to retrieve their staked tokens after requesting a withdrawal. The missing token transfer results in the function only updating internal accounting without performing the required external action, leading to an incomplete withdrawal process and loss of all LST tokens staked in the contract.

## Vulnerability Details

The `_withdraw()` function reduces the share balance of the operator and emits a Withdraw event but does not perform the actual transfer of tokens from the contract to the operator. As a result, the operator will see the reduction in their balance but won't receive any tokens, meaning the withdrawal process remains incomplete.

This issue directly affects the contract's functionality, as operators who wish to withdraw their LST will not be able to do so, locking their funds inside the contract indefinitely.

```Solidity
function _withdraw(address _operator, uint256 _amount) private {
  uint256 sharesAmount = lst.getSharesByStake(_amount);
  shareBalances[_operator] -= sharesAmount;
  totalShares -= sharesAmount;

  //@audit There's no actual transfer of tokens here
  emit Withdraw(_operator, _amount, sharesAmount);
}
```

## POC 
Add the following test in `linkStaking/operator-staking-pool.test.ts`
```javascript
it('withdraw does not transfer tokens', async () => {
    const { signers, accounts, opPool, lst } = await loadFixture(deployFixture)

    //@audit Operator transfers lst tokens to pool
    await lst.transferAndCall(opPool.target, toEther(1000), '0x')
    await lst.connect(signers[1]).transferAndCall(opPool.target, toEther(500), '0x')

    //@audit Operators don't get tokens back on withdrawals. Causing all LST tokens
    // to get stuck in the OperatorStakePool contract.
    const opPoolInitBalance = await lst.balanceOf(opPool);
    await opPool.withdraw(toEther(1000))
    await opPool.connect(signers[1]).withdraw(toEther(200))
    const opPoolEndBalance = await lst.balanceOf(opPool);

    assert.equal(opPoolInitBalance, opPoolEndBalance);
    });

```

## Impact

The consequences of this issue are severe, as it directly affects operator's ability to withdraw their staked tokens. Operators will see their balances decrease after initiating a withdrawal, but they will never actually receive their tokens. This will lead to funds trapped in the contract, making the staked LST tokens inaccessible. All operator's LST stake will be lost.

## Tools Used

Manual analysis 

## Recommendations

A transfer operation in the `_withdraw()` function to move the correct amount of LST tokens from the contract to the operator should be added:\
    &#x20;  &#x20;

```Solidity
function _withdraw(address _operator, uint256 _amount) private {
  uint256 sharesAmount = lst.getSharesByStake(_amount);
  shareBalances[_operator] -= sharesAmount;
  totalShares -= sharesAmount;

  // Perform the actual transfer of tokens to the operator
  lst.safeTransfer(_operator, _amount);

  emit Withdraw(_operator, _amount, sharesAmount);
}

```

