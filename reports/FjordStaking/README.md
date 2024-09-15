# M-01 Flawed handling of Sablier NFT cancellations in fjord staking allows indefinite reward accumulation on refunded assets

## Summary

The Fjord Staking contract does not correctly process the cancellation of Sablier vesting NFTs, enabling users to indefinitely earn rewards on refunded assets from canceled streams. This occurs due to a silent revert in the unstaking process, caused by an incorrect call to `FjordPoints.onUnstake`, allowing rewards to accumulate on funds that have already been refunded.

## Vulnerability Details

The [stakeVested](https://github.com/Cyfrin/2024-08-fjord/blob/0312fa9dca29fa7ed9fc432fdcd05545b736575d/src/FjordStaking.sol#L397-L439) function in the Fjord Staking contract allows users to stake their Sablier vesting NFTs, which hold FJORD tokens, to earn rewards. When a stream is canceled, the `onStreamCanceled` hook is supposed to handle the cancellation by unstaking the refunded assets to stream sender (the unstreamed amount). However, the current implementation always reverts silently due to an incorrect call to `FjordPoints` within `_unstakeVested` execution.

```solidity
function onStreamCanceled(uint256 streamId, address sender, uint128 senderAmount, uint128
    ) external override onlySablier checkEpochRollover {
    // --SNIP
    uint256 amount = uint256(senderAmount) > nftData.amount ? nftData.amount : uint256(senderAmount);
>>>    _unstakeVested(streamOwner, streamId, amount);
    emit SablierCanceled(streamOwner, streamId, sender, amount);
}
```

After the `onStreamCanceled` function handles necessary checks, it calls `_unstakeVested` to unstake the amount of assets being refunded to the stream's sender and reduct that amount from the staked balance. The function `_unstakeVested` calls `points.onUnstaked`, which triggers an underflow revert due to an incorrect `msg.sender` being passed, as shown below:

```solidity
function _unstakeVested(address streamOwner, uint256 _streamID, uint256 amount) internal {
    // --SNIP
>>>    points.onUnstaked(msg.sender, amount); // @audit msg.sender corresdponds to Sablier contract
    emit VestedUnstaked(streamOwner, epoch, amount, _streamID);
}
```

The `msg.sender` refers to `Sablier` and since the `Sablier` contract does not have any stakes, the call reverts with `UnstakingAmountExceedsStakedAmount`:

```solidity
function onUnstaked(address user, uint256 amount) external onlyStaking checkDistribution updatePendingPoints(user) {
    UserInfo storage userInfo = users[user];
    if (amount > userInfo.stakedAmount) {
>>>        revert UnstakingAmountExceedsStakedAmount();
    }

    // --SNIP
}
```

However, the transaction as a whole does not revert because the Sablier version used (v1.0.2) [gracefully handles hook failures](https://github.com/sablier-labs/v2-core/blob/1a206ed69f7b49d229db043e748a098cf3f1da22/src/SablierV2LockupLinear.sol#L432-L438)

```Solidity
function _cancel(uint256 streamId) internal override {
    // --SNIP
    
    // Interactions: if `msg.sender` is the sender and the recipient is a contract, try to invoke the cancel
    // hook on the recipient without reverting if the hook is not implemented, and without bubbling up any
    // potential revert.
    if (msg.sender == sender) {
        if (recipient.code.length > 0) {
            try ISablierV2LockupRecipient(recipient).onStreamCanceled({
                streamId: streamId,
                sender: sender,
                senderAmount: senderAmount,
                recipientAmount: recipientAmount
>>>            }) { } catch { }
        }
    }

    // --SNIP
}
```

As a result, the stream's sender successfully cancels the stream and is refunded, while the staker (recipient) continues earning rewards for the refunded amount indefinitely because his stakes states were not updated.

## Proof Of Concept

The following test case demonstrates how canceling streams will always silently revert the unstaking process on the protocol. Copy and paste the following test case into `test/unit/stakeVested.t.sol`:

```solidity
function test_cancelStreamsNotHandledProperly() public {
        // Setting up the contracts
        FjordPoints fjordPoints = new FjordPoints();
        fjordStaking = new FjordStaking(address(token), address(minter), address(SABLIER), address(this), address(fjordPoints));
        fjordPoints.setStakingContract(address(fjordStaking));
        vm.prank(minter);
        token.approve(address(fjordStaking), 100 ether);


        // Create the stream
        uint amount = 1000 ether;
        deal(address(token), address(this), amount);
        token.approve(address(SABLIER), amount);

        LockupLinear.CreateWithRange memory params;

        params.sender = address(this);
        params.recipient = alice;
        params.totalAmount = uint128(amount);
        params.asset = IERC20(address(token));
        params.cancelable = true;
        params.range = LockupLinear.Range({
            start: uint40(vm.getBlockTimestamp()),
            cliff: uint40(vm.getBlockTimestamp()),
            end: uint40(vm.getBlockTimestamp() + 20 days)
        });
        params.broker = Broker(address(0), ud60x18(0));
        uint streamID = SABLIER.createWithRange(params);

        // Alice stakes her Sablier NFT
        vm.startPrank(alice);
        SABLIER.approve(address(fjordStaking), streamID);
        fjordStaking.stakeVested(streamID);
        vm.stopPrank();

        // The stream is canceled on the same epoch Alice staked
        // The sender will withdraw all the NFT amount since it is canceled just after creation
        vm.prank(address(this));
        SABLIER.cancel(streamID);

        // 3 epochs are passed and rewards are added
        _addRewardAndEpochRollover(10 ether, 3);

        // Even tho the stream was canceled, alice will continue earning rewards
        vm.startPrank(alice);
        fjordStaking.claimReward(false);
        (,uint256 claimAmount) = fjordStaking.claimReceipts(address(alice));
        assertEq(30 ether, claimAmount);

        // Another 4 epochs passed and rewards are added
        _addRewardAndEpochRollover(10 ether, 4);
        vm.startPrank(alice);

        // Complete claim request from last one
        fjordStaking.completeClaimRequest();

        // Alice will keep earning more rewards
        fjordStaking.claimReward(false);
        (, claimAmount) = fjordStaking.claimReceipts(address(alice));
        assertEq(40 ether, claimAmount);

    }
```

**Note**: The contracts were reinitialized at the start of the test case because `FjordStakingBase` [only mocks calls](https://github.com/Cyfrin/2024-08-fjord/blob/0312fa9dca29fa7ed9fc432fdcd05545b736575d/test/FjordStakingBase.t.sol#L53-L71) to the staking contract

To see the silent function revert trace, run the test with `forge test -vvvv --mt test_cancelStreamsNotHandledProperly`:

```bash
[45799] FjordStaking::onStreamCanceled(1402, StakeVestedTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 1000000000000000000000 [1e21], 0)
    │   │   ├─ [24515] 0xB10daee1FCF62243aE27776D7a92D39dC8740f95::transferFrom(FjordStaking: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], alice: [0x328809Bc894f92807417D2dAD6b7C998c1aFdac6], 1402)
    │   │   │   ├─ emit Transfer(from: FjordStaking: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], to: alice: [0x328809Bc894f92807417D2dAD6b7C998c1aFdac6], tokenId: 1402)
    │   │   │   └─ ← ()
>>>    │   │   ├─ [8365] FjordPoints::onUnstaked(0xB10daee1FCF62243aE27776D7a92D39dC8740f95, 1000000000000000000000 [1e21])
>>>    │   │   │   └─ ← UnstakingAmountExceedsStakedAmount()
    │   │   └─ ← UnstakingAmountExceedsStakedAmount()
    │   ├─ emit CancelLockupStream(streamId: 1402, sender: StakeVestedTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], recipient: FjordStaking: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], senderAmount: 1000000000000000000000 [1e21], recipientAmount: 0)
```

## Impact

Canceled streams are not handled correctly, allowing stakers to indefinitely earn rewards on refunded assets of canceled Sablier NFTs

## Tools Used

Manual Review

## Recommendations

Modify the `_unstakeVested` function to pass the `streamOwner` to the `points.onUnstake` instead of `msg.sender`:

```diff
function _unstakeVested(address streamOwner, uint256 _streamID, uint256 amount) internal {
    // --SNIP

-   points.onUnstaked(msg.sender, amount);
+   points.onUnstaked(streamOwner, amount);

    emit VestedUnstaked(streamOwner, epoch, amount, _streamID);
}
```

# M-02 Auction tokens will permanently stuck in `FjordAuctionFactory` if an auction ends with no bids

## Vulnerability Details
The `FjordAuction` contract facilitates auctions where users bid using `FjordPoints` to acquire auction tokens. If an auction concludes with no bids, the auction tokens are transferred to the owner:
```solidity
function auctionEnd() external {
    // --SNIP
    ended = true;
    emit AuctionEnded(totalBids, totalTokens);

    if (totalBids == 0) {
>>>        auctionToken.transfer(owner, totalTokens); 
        return;
    }

    // --SNIP
}
```
In the code above, the tokens are transferred to the `owner`. However, the issue is that the `owner` is set to the `FjordAuctionFactory` contract during deployment, as it is the one that creates the auction contracts. The problem is that the `FjordAuctionFactory` contract, which receives the tokens, has no mechanism to utilize them because creating a new auction always requires transferring tokens from the factory's owner:

```solidity
function createAuction(
    address auctionToken,
    uint256 biddingTime,
    uint256 totalTokens,
    bytes32 salt
) external onlyOwner {
    address auctionAddress = address(
        new FjordAuction{ salt: salt }(fjordPoints, auctionToken, biddingTime, totalTokens)
    );

    // Transfer the auction tokens from the msg.sender to the new auction contract
>>>    IERC20(auctionToken).transferFrom(msg.sender, auctionAddress, totalTokens);

    emit AuctionCreated(auctionAddress);
}
```
As a result, any auction tokens sent back to the factory contract after an auction ends with no bids become stuck and cannot be reused or recovered.

## Impact
If an auction concludes with no bids, the auction tokens are permanently stuck in the `FjordAuctionFactory` contract. This renders the tokens unusable.
## Tools Used
Manual Review

## Recommendations
Consider either transferring the auction tokens to the owner wallet instead of the factory, or utilize them in the `createAuction` as suggested below:
```diff
function createAuction(
    address auctionToken,
    uint256 biddingTime,
    uint256 totalTokens,
    bytes32 salt
) external onlyOwner {
    address auctionAddress = address(
        new FjordAuction{ salt: salt }(fjordPoints, auctionToken, biddingTime, totalTokens)
    );

    // Transfer the auction tokens from the msg.sender to the new auction contract
+   uint factoryBalance = IERC20(auctionToken).balanceOf(address(this));
+   if( factoryBalance > 0) {
+       if (factoryBalance >= totalTokens) IERC20(auctionToken).transferFrom(address(this), auctionAddress, totalTokens);
+       else IERC20(auctionToken).transferFrom(msg.sender, auctionAddress, totalTokens - factoryBalance);
+   }else {
+       IERC20(auctionToken).transferFrom(msg.sender, auctionAddress, totalTokens);
+    }

-    IERC20(auctionToken).transferFrom(msg.sender, auctionAddress, totalTokens);

    emit AuctionCreated(auctionAddress);
}
```