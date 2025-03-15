+++
start-date = "2024-10-7"
end-date = "2024-10-11"
total-time = "15h"
author = "0xSolus"
+++

# Cyfrin Audit Contest Details 
[Diva Protocol]()  

# Findings Summary

| ID     | Title                                                                    | Severity        
| ------ | ------------------------------------------------------------------------ | --------  
| [H-01] | Incorrect Constructor Argument Order in AaveDIVAWrapper Contract.        | High     
| [H-02] | USDT can't be registered as collateralToken                              | High

# Detailed Findings

## H-01 Incorrect Constructor Argument Order in AaveDIVAWrapper Contract.
## Summary

The constructor in `AaveDIVAWrapper` contract, does not use the expected order\
of arguments defined in `AaveDIVAWrapperCore`. Leading to incorrect initialization\
of `_diva` and `_aaveV3Pool` state variables.

## Vulnerability Details

`AaveDIVAWrapper`is initialized with the following arguments:

```soldity
    //@audit Incorrect order of arguments for AaveDIVAWrapperCore.
    // _aaveV3Pool and _diva should swap positions.
    constructor(address _aaveV3Pool, address _diva, address _owner) AaveDIVAWrapperCore(_aaveV3Pool, _diva, _owner) {}
```

But, if we check `AaveDIVAWrapperCore`, the first argument is the address of the `diva_`protocol and the second argument is the address of \`aaveV3Pool\` .

```soldity
    constructor(address diva_, address aaveV3Pool_, address owner_) Ownable(owner_) {
        // Validate that none of the input addresses is zero to prevent unintended initialization with default addresses.
        // Zero address check on `owner_` is performed in the OpenZeppelin's `Ownable` contract.
        if (diva_ == address(0) || aaveV3Pool_ == address(0)) {
            revert ZeroAddress();
        }

        // Store the addresses of DIVA Protocol and Aave V3 in storage.
        _diva = diva_;
        _aaveV3Pool = aaveV3Pool_;
    }

```

# Impact

In first place, this will cause to redeploy the contract as there's no implemented setters to change this state variables. Furthermore, `registerCollateralToken()`will always revert in this case and render the contract unusable. 

## Tools Used

Manual analysis.

## Recommendations

Use the correct order of arguments in the constructor of `AaveDIVAWrapper`.

```Solidity
constructor(address _aaveV3Pool, address _diva, address _owner) AaveDIVAWrapperCore(_diva, _aaveV3Pool, _owner) {}
```


## H-02 USDT can't be registered as collateralToken.
## Summary

The USDT contract doesn't implement the IERC20 interface correctly. In this case functions that are supposed to return a bool (like approve does) don't. 

## Vulnerability Details

When trying to register `USDT `token with `registerCollateralToken()` . This brakes, due to the following line: 

```Solidity
_collateralTokenContract.approve(_aaveV3Pool, type(uint256).max);
```

When trying to call approve, it will revert as IERC20 expects a return `bool` value but the `usdt`contract does not. \
<https://etherscan.io/token/0xdac17f958d2ee523a2206206994597c13d831ec7#code>

## Impact

This issue will make impossible to register `usdt`as a token in the protocol as `registerCollateralToken()` with `usdt` address will always revert.

## Tools Used

Manual analysis.

## Recommendations

In order to mitigate this use `safeIncreaseAllowance` which supports both cases. 

```Solidity
_collateralTokenContract.safeIncreaseAllowance(_aaveV3Pool, type(uint256).max);

```
