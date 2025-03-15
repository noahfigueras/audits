+++
start-date = "2024-12-2"
end-date = "2024-12-7"
total-time = "30h 30m"
author = "0xSolus"
+++

# Audit Contest Details 
[collar-core]()  

# Findings Summary

| ID     | Title                                                                                                            | Severity 
| ------ | ---------------------------------------------------------------------------------------------------------------- | -------- 
| [M-01] | Excess fee to openEscrowLoan is not refunded.                                                                    | Medium   
| [L-01] | Event Spam Vulnerability in createOffer Function Leads to Operational Degradation and Keeper Bot Inefficiencies. | Low      

# Detailed Findings

## [M-1] Excess fee to openEscrowLoan is not refunded.
### Summary
When a loan is opened with an escrow, the escrow requires a fee. In the case that fee amount is over the required fee, the excess is never returned, causing loss of funds for the user that opens a loan.

### Finding Description
A user that wants to open a loan with an escrow through `openEscrowLoan()` has to add a `escrowFee` to startEscrow. On `EscrowSupplierNFT::_startEscrow()` the following line checks that this fee reaches the desired amount.
```solidity
        // we don't check equality to avoid revert due to minor inaccuracies to the upside,
        // even though exact value should be used from the view.
        require(fee >= interestFee(offerId, escrowed), "escrow: insufficient fee");
```
However, the problem here is that, `fee` can be set to any amount and any excess over `interestFee(offerId, escrowed)` should be 
returned to the user. But, instead the full amount will be added as `interestHeld` adding more rewards for the escrowee.
```solidity
    function _startEscrow(uint offerId, uint escrowed, uint fee, uint loanId)
        internal
        returns (uint escrowId)
    {
        require(loansCanOpen[msg.sender], "escrow: unauthorized loans contract");
        // @dev loans is not checked since is directly authed in this contract via setLoansAllowed
        require(configHub.canOpenSingle(asset, address(this)), "escrow: unsupported escrow");

        Offer memory offer = getOffer(offerId);
        require(offer.supplier != address(0), "escrow: invalid offer"); // revert here for clarity

        // check params are supported
        require(configHub.isValidCollarDuration(offer.duration), "escrow: unsupported duration");

        // we don't check equality to avoid revert due to minor inaccuracies to the upside,
        // even though exact value should be used from the view.
        require(fee >= interestFee(offerId, escrowed), "escrow: insufficient fee");

        // check amount
        require(escrowed >= offer.minEscrow, "escrow: amount too low");
        // @dev fee is not taken from offer, because it is transferred in from loans
        uint prevOfferAmount = offer.available;
        require(escrowed <= prevOfferAmount, "escrow: amount too high");

        // storage updates
        offers[offerId].available -= escrowed;
        escrowId = nextTokenId++;
        escrows[escrowId] = EscrowStored({
            offerId: SafeCast.toUint64(offerId),
            loanId: SafeCast.toUint64(loanId),
            expiration: SafeCast.toUint32(block.timestamp + offer.duration),
            released: false, // unset until release
            loans: msg.sender,
            escrowed: escrowed,
            interestHeld: fee, // @audit Full fee amount passed will be set here unnecessarily.
            withdrawable: 0 // unset until release
         });
         ....

```

The functions affected by this issue are `EscrowSupplierNFT::startEscrow()` and `EscrowSupplierNFT::switchEscrow()`.

### Impact Explanation
This will cause loss of funds for the user that opens a loan with a bigger `escrowFee` than required.

### Likelihood Explanation
This issue is likely to occur under the following circumstances:

**Front-End Mismanagement**: Inadequate input validation or sanitization on the front-end may cause users to enter an incorrect escrowFee.

**User Error**: Users might manually input incorrect values when interacting with the contract directly or via third-party tools.

Given these scenarios, the likelihood of occurrence is moderate.


### Proof of Concept
1. User calls openEscrowLoan() with `escrowFee = 2 ETH`.
2. The interestFee = 1 ETH (calculated via `interestFee(offerId, escrowed)`)

The require condition to start an escrow is satisfied because `escrowFee >= interestFee`. However, the full `escrowFee` of 2 ETH is assigned to `interestHeld` instead of 1 ETH. 

The excess 1 ETH is not returned to the user and is instead locked in the contract for the escrow supplier's benefit.

### Recommendation
Pull only the required `escrowFee` from the sender, and store the correct fee value as `interestHeld`. A possible fix could be to check the fee requirements in `startEscrow()` and `switchEscrow()` instead of `_startEscrow()`.

```solidity
    function startEscrow(uint offerId, uint escrowed, uint fee, uint loanId)
        external
        whenNotPaused
        returns (uint escrowId)
    {
        uint _interestFee = interestFee(offerId, escrowed); 
        require(fee >= _interestFee, "escrow: insufficient fee");

        // @dev msg.sender auth is checked vs. loansCanOpen in _startEscrow
        escrowId = _startEscrow(offerId, escrowed, _interestFee, loanId);

        // @dev despite the fact that they partially cancel out, so can be done as just fee transfer,
        // these transfers are the whole point of this contract from product point of view.
        // The transfer events for the full amounts are needed such that the tokens used for the swap
        // in Loans should be "supplier's", and not "borrower's" from CGT tax lows perspective.
        // transfer "borrower's" funds in
        asset.safeTransferFrom(msg.sender, address(this), escrowed + _interestFee);
        // transfer "supplier's" funds out
        asset.safeTransfer(msg.sender, escrowed);
    }

```
```solidity
    function switchEscrow(uint releaseEscrowId, uint offerId, uint newFee, uint newLoanId)
        external
        whenNotPaused
        returns (uint newEscrowId, uint feeRefund)
    {
        Escrow memory previousEscrow = getEscrow(releaseEscrowId);
        // do not allow expired escrow to be switched since 0 fromLoans is used for _endEscrow
        require(block.timestamp <= previousEscrow.expiration, "escrow: expired");

        /*
        1. initially user's escrow "E" secures old ID, "O". O's supplier's funds are away.
        2. E is then "transferred" to secure new ID, "N". N's supplier's funds are taken, to release O.
        3. O is released (with N's funds). N's funds are now secured by E (user's escrow).

        Interest is accounted separately by transferring the full N's interest fee
        (held until release), and refunding O's interest held.
        */

        // "O" (old escrow): Release funds to the supplier.
        // The withdrawable for O's supplier comes from the N's offer, not from Loans repayment.
        // The escrowed loans-funds (E) move into the new escrow of the new supplier.
        // fromLoans must be 0, otherwise escrow will be sent to Loans instead of only the fee refund.
        feeRefund = _endEscrow(releaseEscrowId, previousEscrow, 0);

        // N (new escrow): Mint a new escrow from the offer (can be old or new offer).
        // The escrow funds are funds that have been escrowed in the ID being released ("O").
        // The offer is reduced (which is used to repay the previous supplier)
        // A new escrow ID is minted.
        uint _interestFee = interestFee(offerId, previousEscrow.escrowed); 
        require(newFee >= _interestFee, "escrow: insufficient fee");
        newEscrowId = _startEscrow(offerId, previousEscrow.escrowed, _interestFee, newLoanId);

        // fee transfers
        asset.safeTransferFrom(msg.sender, address(this), _interestFee);
        asset.safeTransfer(msg.sender, feeRefund);

        emit EscrowsSwitched(releaseEscrowId, newEscrowId);
    }

```

## [L-1] Event Spam Vulnerability in createOffer Function Leads to Operational Degradation and Keeper Bot Inefficiencies. 
### Summary
The `createOffer()` function allows anyone to create an escrow or liquidity offer, even with zero value. This can be exploited by malicious actors to spam the contract with fake offers, resulting in an overwhelming number of irrelevant events. This makes it increasingly challenging and computationally expensive for keeper bots to monitor new offers efficiently.

### Finding Description
The `createOffer()` function  in `EscrowSupplierNFT.sol` and `CollarProviderNFT.sol` does not validate the amount parameter to ensure it is greater than zero. This allows any user to create an offer with zero value, generating an event without meaningful economic activity.

Malicious users can repeatedly call this function with zero or negligible values to inflate the number of entries in the liquidityOffers mapping and the associated OfferCreated events. This introduces challenges such as:

**Event Overhead:** Monitoring keeper bots may need to scan a large number of irrelevant events, increasing the time and cost for off-chain processing.

**Storage Bloat:** Although storage costs are borne by the spammer, the overall size of the contract state grows, indirectly increasing the gas cost for other operations involving the mapping.

**Operational Degradation:** Filtering useful offers from spammed events can degrade the efficiency of external services.


### Impact Explanation
This vulnerability:

1. **Increases Event Noise**:
Makes it harder for bots to differentiate legitimate offers from spam.
Potentially causes delays in identifying and acting on new legitimate offers.

2. **Elevates Operational Costs**:
Bots need more computational resources and API calls to retrieve and process larger event logs.
Affects the protocol's ecosystem by degrading performance and trustworthiness of automation.

3. **Indirectly Affects Gas Costs**:
The storage size growth affects the efficiency and gas costs of future operations involving the liquidityOffers mapping.

### Likelihood Explanation
The likelihood of exploitation is moderate to high because:

* The functions are publicly callable and lacks validation on the amount parameter.
* Spamming the functions with zero-value offers incurs minimal gas costs for the attacker while creating significant noise for keeper bots and event listeners.

### Recommendation
To mitigate this vulnerability, implement a validation check in `EscrowSupplierNFT::createOffer()` and `CollarProviderNFT::createOffer()`to ensure that the amount parameter is greater than zero, require a minimum amount or consider restricting the number of offers by address. 

