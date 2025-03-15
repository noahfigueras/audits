+++
start-date = "2025-1-14"
end-date = "NA"
total-time = "NA"
total-rewards = "0"
author = "0xSolus"
+++

# Audit Contest Details 
[Ignite by BENQI]()  

# Findings Summary

| ID     | Title                                                                              | Severity 
| ------ | ---------------------------------------------------------------------------------- | -------- 
| [L-01] | Incorrect Value for SetSlipageUpdated event                                        | Medium   


# Detailed Findings

## L-01 Incorrect Value for SetSlipageUpdated event.
## Summary

In `StakingContract::setSlippage()`the value of  the new slippage is incorrectly added in `SlippageUpdated` event in the case it is out of the requirement range.

## Vulnerability Details

In the case that the new _`_slippage`value is over_ _`maxSlippage`or under_ _`minSlippage`. The value of _`slippage`changes to `boundedSlippage`. However the `SlippageUpdated`event emits the `_slippage`value incorrectly. 

```Solidity
    /**
     * @dev Set the slippage percentage.
     * @param _slippage The new slippage percentage.
     */
    function setSlippage(
        uint256 _slippage
    ) external onlyRole(BENQI_ADMIN_ROLE) {
        uint256 boundedSlippage = _slippage;
        if (_slippage < minSlippage) {
            boundedSlippage = minSlippage;
        } else if (_slippage > maxSlippage) {
            boundedSlippage = maxSlippage;
        }
        uint256 oldSlippage = slippage;
        slippage = boundedSlippage;
        // @audit This _slippage should be replace by slippage. This is because
        // it's updated with boundedSlippage for protection and if _slippage is
        // out of bounds the event emited will be incorrect.
        emit SlippageUpdated(oldSlippage, _slippage);
    }

```

## Impact

Incorrect value added to event. 


