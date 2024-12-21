+++
start-date = "2024-10-29"
end-date = "2024-11-2"
total-time = "16h"
total-rewards = "0 USDC"
author = "0xSolus"
+++

# Sherlock Audit Contest Details 
[Ethos Network Social Contracts ](https://audits.sherlock.xyz/contests/584)  

# Findings Summary

| ID     | Title                                                            | Severity | Status       | Reward     |
| ------ | -----------------------------------------------------------------| -------- | ------------ | ---------- |
| [M-01] | Use call instead of transfer for withdrawals                     | Medium   | Invalid      | 0 USDC     |
| [M-02] | Archives profile should only be allowed to restore their profile | Medium   | Invalid      | 0 USDC     |
| [M-03] | Inconsistent logic for `targetExistsAndAllowedForId()`           | Medium   | Invalid      | 0 USDC     |
| [M-04] | Arbitrary code execution                                         | Medium   | Invalid      | 0 USDC     |


# Detailed Findings
# [M-1] Use call instead of transfer for withdrawals.
`EthosReview::withdrawFunds()` currently uses transfer to send Ether to the `onlyOwner`. 
The transfer function imposes a fixed gas limit of 2300 gas for the transfer, 
which can lead to transaction failures if the recipient is a contract and executes 
some custom logic on `fallback()` or `receive()`.

Gnosis Safe multisig wallets, as well as other smart contract wallets, generally 
require more than 2300 gas due to their internal operations. 

This issue could lead to failed withdrawals by the contract owner due to the gas limit restriction. 
If the transfer fails, the contract owner will be unable to retrieve Ether stored in the contract. 

Note: I'm adding this as medium as team, declared that they are planning to use 
a multisig for `onlyOwner`.

**Recommendation**
To ensure successful withdrawals and avoid issues related to the fixed gas limit, replace transfer with call, which allows specifying a custom gas limit. This approach is more flexible and compatible with smart contract wallets like Gnosis Safe.

```solidity
  /**
   * @dev Withdraws funds.
   * @param paymentToken Payment token address.
   */
  function withdrawFunds(address paymentToken) external onlyOwner {
    if (paymentToken == address(0)) {
        (bool success, ) = msg.sender.call{value: address(this).balance}("");
        require(success, "Ether transfer failed");
    } else {
      IERC20(paymentToken).transfer(msg.sender, IERC20(paymentToken).balanceOf(address(this)));
    }
  }


```

# [M-2] Archives profile should only be allowed to restore their profile.
According to the team, archive profiles should only allow to restore their profile.
But, this invariant is broken in multiple scenarios.

The following functions are not restricted to archive profiles:
1. `EthosProfile::uninvateUser()`-> allows to uninvitate a user.
2. `EthosReview::archiveReview()` -> allows to archives a review.
```javascript
  // test/EthosProfile.test.ts
  describe('archive profiles should only be able to restore their profile', () => {
    it('uninviteUser', async () => {
      const { ethosProfile, PROFILE_CREATOR_0, OWNER } =
        await loadFixture(deployFixture);

      await ethosProfile.connect(OWNER).inviteAddress(PROFILE_CREATOR_0.address);
      await ethosProfile.connect(OWNER).archiveProfile();

      await expect(ethosProfile.connect(OWNER).uninviteUser(PROFILE_CREATOR_0.address)).to.be.reverted;
    });
    it('archiveReview', async () => {
      const { ethosProfile, ethosReview, PROFILE_CREATOR_0, OWNER } =
        await loadFixture(deployFixture);

      await ethosProfile.connect(OWNER).inviteAddress(PROFILE_CREATOR_0.address);
      await ethosReview.connect(OWNER).addReview(  
          0,
          '0xD6d547791DF4c5f319F498dc2d706630aBE3e36f',
          '0x0000000000000000000000000000000000000000',
          'this is a comment',
          'this is metadata',
          { account: '', service: '' },
        );
      await ethosProfile.connect(OWNER).archiveProfile();
      await expect(ethosReview.connect(OWNER).archiveReview(0)).to.be.reverted;
    });
  });
```

3. `EthosAttestation::claimAttestation()` -> allows only to claim previously created Attestation.
4. `EthosAttestation::archiveAttestation()` -> allows to archive an attestation.
```javascript
  // test/EthosAttestations.test.ts
  describe('archive profiles should only be able to restore their profile', () => {
    it('previously created claimAttestation', async () => {
      const {
        OWNER,
        PROFILE_CREATOR_0,
        ethosProfile,
        ethosAttestation,
        SERVICE_X,
        ACCOUNT_NAME_BEN,
        ATTESTATION_EVIDENCE_0,
        EXPECTED_SIGNER,
      } = await loadFixture(deployFixture);

      await ethosProfile.connect(OWNER).inviteAddress(PROFILE_CREATOR_0.address);
      await ethosProfile.connect(PROFILE_CREATOR_0).createProfile(1);
      await ethosProfile.connect(OWNER).archiveProfile();
      const creator0profileId = String(
        await ethosProfile.profileIdByAddress(PROFILE_CREATOR_0.address),
      );
      const ownerprofileId = String(
        await ethosProfile.profileIdByAddress(OWNER.address),
      );
      const randValue = '123';

      const signature = await common.signatureForCreateAttestation(
        creator0profileId,
        randValue,
        ACCOUNT_NAME_BEN,
        SERVICE_X,
        ATTESTATION_EVIDENCE_0,
        EXPECTED_SIGNER,
      );
      const signatureOwner = await common.signatureForCreateAttestation(
        ownerprofileId,
        randValue,
        ACCOUNT_NAME_BEN,
        SERVICE_X,
        ATTESTATION_EVIDENCE_0,
        EXPECTED_SIGNER,
      );

      const attestationHash = await ethosAttestation.getServiceAndAccountHash(
        SERVICE_X,
        ACCOUNT_NAME_BEN,
      );

      expect(
        await ethosAttestation.getAttestationHashesByProfileId(creator0profileId),
      ).to.deep.equal([], 'Wrong attestationHashesByProfileId before');

      await ethosAttestation
        .connect(PROFILE_CREATOR_0)
        .createAttestation(
          creator0profileId,
          randValue,
          { account: ACCOUNT_NAME_BEN, service: SERVICE_X },
          ATTESTATION_EVIDENCE_0,
          signature,
        );

      expect(
        await ethosAttestation.getAttestationHashesByProfileId(creator0profileId),
      ).to.deep.equal([attestationHash], 'Wrong attestationHashesByProfileId after');

      await ethosAttestation
        .connect(OWNER)
        .createAttestation(
          ownerprofileId,
          randValue,
          { account: ACCOUNT_NAME_BEN, service: SERVICE_X },
          ATTESTATION_EVIDENCE_0,
          signatureOwner,
        );

      expect(
        await ethosAttestation.getAttestationHashesByProfileId(ownerprofileId),
      ).to.deep.equal([attestationHash], 'Wrong attestationHashesByProfileId after');
    });
    it('allows to archive an attestation', async () => {
      const {
        OWNER,
        PROFILE_CREATOR_0,
        ethosProfile,
        ethosAttestation,
        SERVICE_X,
        ACCOUNT_NAME_BEN,
        ATTESTATION_EVIDENCE_0,
        EXPECTED_SIGNER,
      } = await loadFixture(deployFixture);

      await ethosProfile.connect(OWNER).inviteAddress(PROFILE_CREATOR_0.address);

      const ownerprofileId = String(
        await ethosProfile.profileIdByAddress(OWNER.address),
      );
      const randValue = '123';

      const signatureOwner = await common.signatureForCreateAttestation(
        ownerprofileId,
        randValue,
        ACCOUNT_NAME_BEN,
        SERVICE_X,
        ATTESTATION_EVIDENCE_0,
        EXPECTED_SIGNER,
      );

      const attestationHash = await ethosAttestation.getServiceAndAccountHash(
        SERVICE_X,
        ACCOUNT_NAME_BEN,
      );

      await ethosAttestation
        .connect(OWNER)
        .createAttestation(
          ownerprofileId,
          randValue,
          { account: ACCOUNT_NAME_BEN, service: SERVICE_X },
          ATTESTATION_EVIDENCE_0,
          signatureOwner,
        );

      expect(
        await ethosAttestation.getAttestationHashesByProfileId(ownerprofileId),
      ).to.deep.equal([attestationHash], 'Wrong attestationHashesByProfileId after');

      await ethosProfile.connect(OWNER).archiveProfile();
      await ethosAttestation.connect(OWNER).archiveAttestation(attestationHash);
      const attestation = await ethosAttestation.connect(OWNER).getAttestationByHash(attestationHash);

      expect(attestation.archived).to.be.eq(true);
    });
  });
```

**Recommendation:**
Consider restricting those functions to archive profiles. Using `verifiedProfileIdForAddress(addr)`
will revert in the case of an archive profile.

# [M-3] Inconsistent logic for `targetExistsAndAllowedForId()`
The logic of this function is inconsistent with it's comments throughout multiple
contracts.

The function only checks if `targetId` exists and if exists it will be allowed. But, this is not 
the case for scenarios where `targetId` is archived, in that scenario even if it exists
that id, should not be allowed. 

Contracts affected:  
1. `EthosProfile::targetExistsAndAllowdForId()`
2. `EthosReview::targetExistsAndAllowdForId()`
3. `EthosAttestation::targetExistsAndAllowdForId()`

**Recommendation**
This is a specific fix for `EthosProfile::targetExistsAndAllowdForId()`, but this 
idea could be implemented for the other contracts affected.
```solidity
  // ITargetStatus implementation
  /**
   * @dev Checks whether profile verified & is allowed to be used.
   * @param targetId Profile id.
   * @return exist Whether profile verified.
   * @return allowed Whether profile is allowed to be used.
   * @notice This is a standard function used across Ethos contracts to validate profiles.
   */
  function targetExistsAndAllowedForId(
    uint256 targetId
  ) external view returns (bool exist, bool allowed) {
    Profile storage profile = profiles[targetId];

    exist = profile.createdAt > 0;
    allowed = !profile.archived;
  }

```

# [M-4] Arbitrary code execution. 
Any verified profile can execute arbitrary code in behalf of `EthosDiscussion` 
as `msg.sender` through `EthosDiscussion::addReply()`.

Whenever a verified profile tries to add a reply through `EthosDiscussion::addReply()`
and `targetContract` is not `EthosDiscussion` address. Then `_checkIfTargetExistsAndAllowed(targetContract, targetId)`
is executed.
```solidity
  /**
   * @dev Checks if the target exists and is allowed.
   * @param targetContract Target contract address.
   * @param targetId Target id.
   */
  function _checkIfTargetExistsAndAllowed(address targetContract, uint256 targetId) private view {
    (bool exists, ) = ITargetStatus(targetContract).targetExistsAndAllowedForId(targetId);

    if (!exists) {
      revert TargetNotFound(targetContract, targetId);
    }
  }
```
The problem here is that `targetContract` is not validated and can be any contract.
Therefore, if a bad actor deploys a contract that implements `ITargetStatus`, the 
code will be executed with `EthosDiscussion` as `msg.sender`.

There's is no immediate impact by this action in this contract, as `EthosDiscussion`
doesn't execute any permissioned functions in the scoped contracts. Anyway, I'm 
reporting this issue as medium as the contract is upgradable and can cause issues 
in future updates. 

**Recommendation:**
Consider using `contractAddressManager.checkIsEthosContract(target)` or similar 
to validate the contracts.


