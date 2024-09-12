# H-01 Not upadting `_totalAuctionTokenAllocation` when removing last auction config at cooldown leads to wrong accounting of `_totalAuctionTokenAllocation` and permanent lock of auction tokens

## Vulnerability Details
[removeAuctionConfig](https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L107-L133) is used to remove auction config set for last epoch. If the auction was started but the cooldown has not passed yet, both auction config and epoch info are deleted and the `_currentEpochId` is decremented:
```solidity
function removeAuctionConfig() external override onlyDAOExecutor {
    /// only delete latest epoch if auction is not started
    uint256 id = _currentEpochId;
        
    EpochInfo storage info = epochs[id];
    // ...

    bool configSetButAuctionStartNotCalled = auctionConfigs[id+1].duration > 0;
    if (!configSetButAuctionStartNotCalled) {
        /// @dev unlikely because this is a DAO execution, but avoid deleting old ended auctions
        if (info.hasEnded()) { revert AuctionEnded(); }
        /// auction was started but cooldown has not passed yet
        delete auctionConfigs[id];
        delete epochs[id];
        _currentEpochId = id - 1;
        // @audit NOT UPDATING `_totalAuctionTokenAllocation`
        emit AuctionConfigRemoved(id, id);
    } 
    // ...
}
```
The issue is that `_totalAuctionTokenAllocation` is not updated in this situation and this will lead to the following consequences for the next epochs:

1. `_totalAuctionTokenAllocation` will always contain extra auction tokens for all subsequent epochs.
2. Those extra auction tokens can not be recovered.

Consider the following example (that is demonstrated by PoC below) to better explain the consequences:
- For simplicity, we set the auction config for the first epoch with `100 ether` auction tokens sent to the Spice contract
- The auction is started, and based on the [startAuction logic](https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L158-L173), the following states are updates as follows:
    - `info.totalAuctionTokenAmount` = 100 ether:
    ```solidity
        uint256 totalAuctionTokenAllocation = _totalAuctionTokenAllocation[auctionToken]; // @audit  = 0
        uint256 balance = IERC20(auctionToken).balanceOf(address(this)); // @audit 100 ether
        uint256 epochAuctionTokenAmount = balance - (totalAuctionTokenAllocation - _claimedAuctionTokens[auctionToken]); // @audit 100 ether
        // ...
        info.totalAuctionTokenAmount = epochAuctionTokenAmount; // @audit 100 ether
    ```
    - `_totalAuctionTokenAllocation[auctionToken]` = 100 ether:
    ```solidity
        _totalAuctionTokenAllocation[auctionToken] = totalAuctionTokenAllocation + epochAuctionTokenAmount;
    ```
- Now, for some reason, the DaoExecutor removes the auction config for the last epoch (the first epoch in our example). Because the epoch is still at cooldown, the following block of [removeAuctionConfig](https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L119-L127) will be executed:
    ```solidity
        delete auctionConfigs[id];
        delete epochs[id];
        _currentEpochId = id - 1;
    ```
    Notice that `_totalAuctionTokenAllocation` is not updated.
- Now, the DaoExecutor wants to set up a new auction (with `100 ether` auction token amount) and starts it, so he calls [setAuctionConfig](https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L84-L104) and calls [startAuction](https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L138-L176). **Normally, we should not feed the Spice contract with `100 ether` because the contract already holds it from the previous deleted auction**. However, because the `_totalAuctionTokenAllocation` still holds `100 ether`, the `epochAuctionTokenAmount` will be evaluated to zero, and thus the transaction will revert:
    ```solidity
        uint256 totalAuctionTokenAllocation = _totalAuctionTokenAllocation[auctionToken]; // @audit 100 ether
        uint256 balance = IERC20(auctionToken).balanceOf(address(this)); // @audit 100 ether
        uint256 epochAuctionTokenAmount = balance - (totalAuctionTokenAllocation - _claimedAuctionTokens[auctionToken]); // @audit = 0

        if (epochAuctionTokenAmount < config.minimumDistributedAuctionToken) { revert NotEnoughAuctionTokens(); } // @audit tx will revert
    ```
    So, the contract should be fed with an extra 100 ether if we want to start a new auction with 100 ether tokens.
- Now, we fed the Spice contract with the extra 100 ether that we were obliged to, the `_totalAuctionTokenAllocation` will be updated wrongly:
    ```solidity
        uint256 totalAuctionTokenAllocation = _totalAuctionTokenAllocation[auctionToken]; // @audit 100 ether
        uint256 balance = IERC20(auctionToken).balanceOf(address(this)); // @audit 200 ether
        uint256 epochAuctionTokenAmount = balance - (totalAuctionTokenAllocation - _claimedAuctionTokens[auctionToken]); // @audit 100 ether
        // ...

        info.totalAuctionTokenAmount = epochAuctionTokenAmount; // @audit 100 ether
        // Keep track of total allocation auction tokens per epoch
        _totalAuctionTokenAllocation[auctionToken] = totalAuctionTokenAllocation + epochAuctionTokenAmount; // @audit 200 ether !!!!!
    ```
    We can see that the `_totalAuctionTokenAllocation[auctionToken]` is updated to `200 ether` while it should have been updated to `100 ether` because there is only ONE auction running with 100 ether tokens.
- Even if the DaoExecutor attempts to recover those extra 100 ether tokens stucking at contract after the auction ends, he will not be able to do so. For simplicity, let's consider no one claimed his tokens yet:
    ```solidity
    function recoverToken(
        address token,
        address to,
        uint256 amount
    ) external override onlyDAOExecutor {
        // ...

        uint256 epochId = _currentEpochId;
        EpochInfo storage info = epochs[epochId];
        // ....
        uint256 totalAuctionTokenAllocation = _totalAuctionTokenAllocation[token]; // @audit 200 ether
        uint256 balance = IERC20(token).balanceOf(address(this)); // @audit 200 ether
        uint256 maxRecoverAmount = balance - (totalAuctionTokenAllocation - _claimedAuctionTokens[token]); // @audit = 0
        
        if (amount > maxRecoverAmount) { revert CommonEventsAndErrors.InvalidParam(); } // @audit tx will revert

    }
    ```
### PoC
The following PoC demonstrates the example illustrated above. Copy/Paste the following code into `test/forge/templegold/SpiceAuction.t.sol`:
```solidity
contract SpiceAuctionAccessTest is SpiceAuctionTestBase {

    error NotEnoughAuctionTokens();
    error InvalidParam();

    // ...
function testPoC() public {
    // templeGold is the auction token, with 100 ether amount balance for spice contract
    _startAuction(true, true);
    address auctionToken = spice.getAuctionTokenForCurrentEpoch();
    vm.startPrank(daoExecutor);
    spice.removeAuctionConfig(); // removing the latest epoch config before it starts
        

    uint256 auctionTokenBalance = IERC20(auctionToken).balanceOf(address(spice));
    assertEq(100 ether, auctionTokenBalance);
    // Now, we should start the next epoch, BUT WE DO NOT NEED TO SEND AUCTION TOKENS TO THE SPICE CONTRACT
    // Because the contract already has 100 ether balance.
    // But, if we attempt to start the auction without feeding the contract -> it will revert 

    ISpiceAuction.SpiceAuctionConfig memory config = _getAuctionConfig();
    spice.setAuctionConfig(config);
    if (config.starter != address(0)) { vm.startPrank(config.starter); }
    vm.warp(block.timestamp + config.waitPeriod);
    vm.expectRevert(abi.encodeWithSelector(NotEnoughAuctionTokens.selector)); // Revert message
    spice.startAuction();

    // So, we are obliged to feed to the contract with other auction tokens
    auctionToken = config.isTempleGoldAuctionToken ? address(templeGold) : spice.spiceToken();
    dealAdditional(IERC20(auctionToken), address(spice), 100 ether);

    // Now, the contract holds 200 ether of auction token (extra 100 ether were obliged to be added)
    auctionTokenBalance = IERC20(auctionToken).balanceOf(address(spice));
    assertEq(200 ether, auctionTokenBalance);

    uint256 epoch = spice.currentEpoch();
    IAuctionBase.EpochInfo memory epochInfo = spice.getEpochInfo(epoch);
    uint64 startTime = uint64(block.timestamp + config.startCooldown);
    uint64 endTime = uint64(startTime+config.duration);
    vm.expectEmit(address(spice));
    emit AuctionStarted(epoch+1, alice, startTime, endTime, 100 ether); // Notice that only 100 ether were counted for the next epoch
    spice.startAuction(); 
    vm.startPrank(daoExecutor);

    // attempting to recover those extra 100 ether after the auction ends.
    vm.warp(endTime);
    vm.expectRevert(abi.encodeWithSelector(InvalidParam.selector));
    spice.recoverToken(auctionToken, alice, 100 ether);

    // 100 ether of auction token are locked permanently in the contract
}
}
```
```bash
forge test --mt testPoC
[PASS] testPoC() (gas: 734490)
```

## Impact
If the last auction config at cooldown is removed, two serious impacts will occur:

1. `_totalAuctionTokenAllocation` will always contain extra auction tokens for all subsequent epochs.
2. Those extra auction tokens can not be recovered.
## Tools Used
Manual Review
## Recommendations
Consider updating `_totalAuctionTokenAllocation` when removing the config of the last started auction at the cooldown period. Below is a suggestion for an updated code of `SpiceAuction::removeAuctionConfig`:
```diff
function removeAuctionConfig() external override onlyDAOExecutor {
    /// only delete latest epoch if auction is not started
    uint256 id = _currentEpochId;
        
    EpochInfo storage info = epochs[id];
    // _currentEpochId = 0
    if (info.startTime == 0) { revert InvalidConfigOperation(); }
    // Cannot reset an ongoing auction
    if (info.isActive()) { revert InvalidConfigOperation(); }
    /// @dev could be that `auctionStart` is triggered but there's cooldown, which is not reached (so can delete epochInfo for _currentEpochId)
    // or `auctionStart` is not triggered but `auctionConfig` is set (where _currentEpochId is not updated yet)
    bool configSetButAuctionStartNotCalled = auctionConfigs[id+1].duration > 0;
    if (!configSetButAuctionStartNotCalled) {
        /// @dev unlikely because this is a DAO execution, but avoid deleting old ended auctions
        if (info.hasEnded()) { revert AuctionEnded(); }
        /// auction was started but cooldown has not passed yet
+        SpiceAuctionConfig storage config = auctionConfigs[id];
+        (,address auctionToken) = _getBidAndAuctionTokens(config);
+        _totalAuctionTokenAllocation[auctionToken] -= info.totalAuctionTokenAmount;
        delete auctionConfigs[id];
        delete epochs[id];
        _currentEpochId = id - 1;
        emit AuctionConfigRemoved(id, id);
    } else {
        // `auctionStart` is not triggered but `auctionConfig` is set
        id += 1;
        delete auctionConfigs[id];
        emit AuctionConfigRemoved(id, 0);
    }
}
```

# L-01 No TGLD recover mechanism in `DaiGoldAuction` when the auction ends without any bid

## Vulnerability Details
In `DaiGoldAuction` contract, when an auction ends with no bids, the TGLD tokens are locked into the contract and can’t be recovered. [recoverToken](https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/DaiGoldAuction.sol#L260-L294) can not be used to recover TGLD tokens because it will [revert](https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/DaiGoldAuction.sol#L279) when the auction epoch ends:
```solidity
function recoverToken(
    address token,
    address to,
    uint256 amount
) external override onlyElevatedAccess {
    // ...

    if (info.hasEnded()) { revert AuctionEnded(); }

    // ...
}
```
There is no other way to recover the auction tokens. As a consequence, the TGLD tokens will be permanently locked in the contract
## Impact
- If the auction epoch ends without any bid, the TGLD tokens will be permanently locked in the contract
## Tools Used
Manual Review
## Recommendations
Consider adding a separate function that recovers TGLD tokens when the auction epoch ends without any bids. Similar to the [one implemented in SpiceAuction::recoverAuctionTokenForZeroBidAuction](https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L275-L292)

# L-02 Auction tokens that were mistakenly sent to the Spice contract in the first auction epoch can not be recovered

## Vulnerability Details
In `SpiceAuction`, [recoverToken](https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L234-L268) function is meant to be used to recover auction tokens that were mistakenly sent to the contract for last but not started auction:
```solidity
function recoverToken(
    address token,
    address to,
    uint256 amount
) external override onlyDAOExecutor {
    if (to == address(0)) { revert CommonEventsAndErrors.InvalidAddress(); }
    if (amount == 0) { revert CommonEventsAndErrors.ExpectedNonZero(); }

    if (token != spiceToken && token != templeGold) {
        emit CommonEventsAndErrors.TokenRecovered(to, token, amount);
        IERC20(token).safeTransfer(to, amount);
        return;
    }
    uint256 epochId = _currentEpochId;
    EpochInfo storage info = epochs[epochId];
    /// @dev use `removeAuctionConfig` for case where `auctionStart` is called and cooldown is still pending
    if (info.startTime == 0) { revert InvalidConfigOperation(); }
    if (!info.hasEnded() && auctionConfigs[epochId+1].duration == 0) { revert RemoveAuctionConfig(); } // <------------------------------------
    
    // ...
    IERC20(token).safeTransfer(to, amount);
}
```
From the code snippet, more particularly in this check:
```solidity
// @audit if `auctionConfigs[epochId+1].duration == 0`, it means we are the latest, unfinished auction epoch
if (!info.hasEnded() && auctionConfigs[epochId+1].duration == 0) { revert RemoveAuctionConfig(); }
```
we can see that if we are recovering auction tokens and the last auction epoch is at cooldown, `removeAuctionConfig` should be used first to remove the configuration of the last epoch.

Below is the code of [removeAuctionConfig](https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L107-L133):
```solidity
function removeAuctionConfig() external override onlyDAOExecutor {
    /// only delete latest epoch if auction is not started
    uint256 id = _currentEpochId;
        
    EpochInfo storage info = epochs[id];
    // _currentEpochId = 0
    if (info.startTime == 0) { revert InvalidConfigOperation(); }
    // Cannot reset an ongoing auction
    if (info.isActive()) { revert InvalidConfigOperation(); }
    /// @dev could be that `auctionStart` is triggered but there's cooldown, which is not reached (so can delete epochInfo for _currentEpochId)
    // or `auctionStart` is not triggered but `auctionConfig` is set (where _currentEpochId is not updated yet)
    bool configSetButAuctionStartNotCalled = auctionConfigs[id+1].duration > 0;
    if (!configSetButAuctionStartNotCalled) {
        /// @dev unlikely because this is a DAO execution, but avoid deleting old ended auctions
        if (info.hasEnded()) { revert AuctionEnded(); }
        /// auction was started but cooldown has not passed yet
        delete auctionConfigs[id];
        delete epochs[id];
        _currentEpochId = id - 1;
        emit AuctionConfigRemoved(id, id);
     } else {
        // `auctionStart` is not triggered but `auctionConfig` is set
        id += 1;
        delete auctionConfigs[id];
        emit AuctionConfigRemoved(id, 0);
    }
}
```
Because we are removing the config of the latest auction at cooldown, the following part of code will be executed:
```solidity
if (!configSetButAuctionStartNotCalled) {
    /// @dev unlikely because this is a DAO execution, but avoid deleting old ended auctions
    if (info.hasEnded()) { revert AuctionEnded(); }
    /// auction was started but cooldown has not passed yet
    delete auctionConfigs[id];
    delete epochs[id]; // @audit Notice that we delete the epoch 
    _currentEpochId = id - 1;
    emit AuctionConfigRemoved(id, id);
}
```
We can see that the function deletes the epoch and **decrements the `epochId`**. 

The issue is that the `recoverToken` function does not handle properly the recover process for the first auction, i.e when there is only auction. Consider the following example:
- We have one started epoch with `100 ether` auction tokens, it is at cooldown.
- `2 ether` were mistakenly sent to the contract during cooldown. The DaoExecutor attempts to recover them by calling `recoverToken`
- Given that we are at the latest auction at cooldown, the config must be removed first:
    ```solidity
    if (!info.hasEnded() && auctionConfigs[epochId+1].duration == 0) { revert RemoveAuctionConfig(); } 
    ```
    So, the DaoExecutor calls `removeAuctionConfig`. Notice that after the tx, `epochId` will be decremented to 0.
- Now, if we attempt to recover back the auction tokens, the transaction will still revert, but this time because of [invalid epoch](https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L251):
    ```solidity
    EpochInfo storage info = epochs[epochId];
    /// @dev use `removeAuctionConfig` for case where `auctionStart` is called and cooldown is still pending
    if (info.startTime == 0) { revert InvalidConfigOperation(); }
    ```
- If the new epoch (with id 1) is started, that `2 ether` will be counted for the epoch auction tokens amount. And thus, can not be recovered:
    ```solidity
    // function: startAuction

    uint256 balance = IERC20(auctionToken).balanceOf(address(this));
    uint256 epochAuctionTokenAmount = balance - (totalAuctionTokenAllocation - _claimedAuctionTokens[auctionToken]);
    ```
    
**As a consequence, tokens mistakenly sent to the first epoch during the cooldown period can not be recovered**.


## Impact
Auction tokens that were mistakenly sent to the first epoch during the cooldown period can not be recovered.

## Tools Used
Manual Review
## Recommendations
Consider handling the case of recovering auction tokens for the first epoch. Below is a suggestion for an updated code of `recoverToken`:

```diff
function recoverToken(
        address token,
        address to,
        uint256 amount
    ) external override onlyDAOExecutor {
        if (to == address(0)) { revert CommonEventsAndErrors.InvalidAddress(); }
        if (amount == 0) { revert CommonEventsAndErrors.ExpectedNonZero(); }

        if (token != spiceToken && token != templeGold) {
            emit CommonEventsAndErrors.TokenRecovered(to, token, amount);
            IERC20(token).safeTransfer(to, amount);
            return;
        }

        uint256 epochId = _currentEpochId;
        EpochInfo storage info = epochs[epochId];
        /// @dev use `removeAuctionConfig` for case where `auctionStart` is called and cooldown is still pending
        if (info.startTime == 0) { revert InvalidConfigOperation(); }
        
+        if (!info.isActive() && !info.hasEnded() && epochId == 1) {
+            uint256 balance = IERC20(token).balanceOf(address(this));
+            uint256 recoverAmount = balance - totalAuctionTokenAllocation;
+            IERC20(token).safeTransfer(to, recoverAmount);
+            emit CommonEventsAndErrors.TokenRecovered(to, token, recoverAmount);
+            return;
+        }
        if (!info.hasEnded() && auctionConfigs[epochId+1].duration == 0) { revert RemoveAuctionConfig(); } // audit-finding Contradiction?
        

        /// @dev Now `auctionStart` is not triggered but `auctionConfig` is set (where _currentEpochId is not updated yet)
    
        // check to not take away intended tokens for claims
        // calculate auction token amount
        uint256 totalAuctionTokenAllocation = _totalAuctionTokenAllocation[token];
        uint256 balance = IERC20(token).balanceOf(address(this));
        uint256 maxRecoverAmount = balance - (totalAuctionTokenAllocation - _claimedAuctionTokens[token]);
        
        if (amount > maxRecoverAmount) { revert CommonEventsAndErrors.InvalidParam(); } 
        
        IERC20(token).safeTransfer(to, amount);

        emit CommonEventsAndErrors.TokenRecovered(to, token, amount);
    }
```
