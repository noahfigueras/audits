
# Audit Contest Details 
[Chakra]()  


# Findings Summary

| ID     | Title                                                                       | Severity 
| ------ | --------------------------------------------------------------------------- | -------- 
| [H-01] | Transaction Front-running can cause funds to be lost/locked on origin chain | High     
| [H-02] | Funds lost/locked on failed transaction in Handler Contract                 | High     

# Detailed Findings

## [H-1] Transaction Front-running can cause funds to be lost/locked on origin chain.
Once a transaction is created in the origin chain and send to chakra validators with `send_cross_chain_msg()`.
Chakra validators are supposed to sign it and send it to the destination chain with `receive_cross_chain_msg()`.
Even though the previous function checks for the tx to be signed for whitelisted validators. The transaction 
can be front-run as that function doesn't have any access controls and allows anyone to call it.  

The problem here is not caused by the front-run transaction in itself, because that one will 
go through and perform the necessary actions on the destination chain. The problem arises because the
original transaction that was front-run will fail due to replay protection in the following 
[line](https://github.com/code-423n4/2024-08-chakra/blob/main/solidity/settlement/contracts/ChakraSettlement.sol#L200). 
As the original tx was front-run `CrossChainMsgStatus` will change and the validators transaction will 
always revert with `Invalid transaction status`.

Once this scenario happens, if the chakra validator reports back to the origin chain with
`CrossChainMsgStatus.Failed` calling `receive_cross_chain_callback()`. This will flag that specific txid with a `failed` status.
This becomes a problem because another validator that picked up the event of the front-run 
tx that was validated successfully. When he tries to call `receive_cross_chain_callback()`
with the same txid that was successful on the destination chain, it always will revert
with `Invalid transaction status` in [line](https://github.com/code-423n4/2024-08-chakra/blob/main/solidity/settlement/contracts/ChakraSettlement.sol#L313).

This issue will cause tokens to be burn, minted on the destination chain, but not 
on the origin chain.

**Recommendation:**
Implement Access Controls for `onlyValidators`.
```solidity
function receive_cross_chain_msg(
    uint256 txid,
    string memory from_chain,
    uint256 from_address,
    uint256 from_handler,
    address to_handler,
    PayloadType payload_type,
    bytes calldata payload,
    uint8 sign_type, // validators signature type /  multisig or bls sr25519
    bytes calldata signatures // signature array
    ) external onlyValidator {
  ...
}
```

## [H-2] Funds lost/locked on failed transaction in Handler Contract 
In `ChakraSettlmentHandler.sol` after erc20 tx `cross_chain_erc20_settlement()` is sent
successfully. If something on the destination chain fails, due to lack of erc20 funds on the 
other chain or for some other reason the transaction is flaged as failed. Tokens in the origin
chain will get lost or locked.

On `ChakraSettlmentHandler.sol::receive_cross_chain_callback()` no action is performed to undo
the transaction in case of failure [here](https://github.com/code-423n4/2024-08-chakra/blob/main/solidity/handler/contracts/ChakraSettlementHandler.sol#L391). 
In this case, if `mode == SettlementMode.BurnUnlock` all funds will be lost as the burn action can't be reverted. In any other modes, all funds
will remain locked on the contract.

**Recommendation:**
1. Consider locking tokens instead of burning them directly.
```solidity
function cross_chain_erc20_settlement(
    string memory to_chain,
    uint256 to_handler,
    uint256 to_token,
    uint256 to,
    uint256 amount
    ) external {
  require(amount > 0, "Amount must be greater than 0");
  require(to != 0, "Invalid to address");
  require(to_handler != 0, "Invalid to handler address");
  require(to_token != 0, "Invalid to token address");

  // @audit Consider locking all funds until callback is verififed.
  _erc20_lock(msg.sender, address(this), amount);

  ...
}

```

2. Considering handling the undo of the transaction in the case of failure.
```solidity
/**
 * @dev Receives a cross-chain callback
 * @param txid The transaction ID
 * @param from_chain The source chain
 * @param from_handler The source handler
 * @param status The status of the cross-chain message
 * @return bool True if successful, false otherwise
 */
function receive_cross_chain_callback(
    uint256 txid,
    string memory from_chain,
    uint256 from_handler,
    CrossChainMsgStatus status,
    uint8 /* sign_type */, // validators signature type /  multisig or bls sr25519
    bytes calldata /* signatures */
    ) external onlySettlement returns (bool) {
  //  from_handler need in whitelist
  if (is_valid_handler(from_chain, from_handler) == false) {
    return false;
  }

  require(
      create_cross_txs[txid].status == CrossChainTxStatus.Pending,
      "invalid CrossChainTxStatus"
      );

  if (status == CrossChainMsgStatus.Success) {
    if (mode == SettlementMode.MintBurn) {
      _erc20_burn(address(this), create_cross_txs[txid].amount);
    } else if (mode == SettlementMode.BurnUnlock) {
      // @audit Consider burn tokens once verified.
      _erc20_burn(address(this), create_cross_txs[txid].amount);
    }


    create_cross_txs[txid].status = CrossChainTxStatus.Settled;
  }

  if (status == CrossChainMsgStatus.Failed) {
      // @audit Consider handle transaction failure accordingly.
      _erc20_unlock(create_cross_txs[txid].from, create_cross_txs[txid].amount);
      create_cross_txs[txid].status = CrossChainTxStatus.Failed;
    }

    return true;
  }
  ```


