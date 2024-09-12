# H-01 Incorrect check to verify if plotId exceeds available plots in `_farmPlots`

## Impact
The [_farmPlots](https://github.com/code-423n4/2024-07-munchables/blob/94cf468aaabf526b7a8319f7eba34014ccebe7b9/src/managers/LandManager.sol#L232-L310) function is executed during various operations on a plot, such as staking, unstaking, and transferring tokens to a new plot. This function attempts to verify whether the plotId exceeds the number of available plots for each staked token:
```solidity
function _farmPlots(address _sender) internal {
    // ...

    if (_getNumPlots(landlord) < _toiler.plotId) {
        timestamp = plotMetadata[landlord].lastUpdated;
        toilerState[tokenId].dirty = true;
    }

    // ...
}
```

The issue is that the check is incorrect because `plotId` counting starts from zero, the highest valid plot id is `available plots - 1`, while the current check considers plot with id equal to `available plots` valid . If for example, the total available plots is *one*, the only valid plotId is *0*, or if the total available plots is `2` the only valid plots are `0` and `1`. This is confirmed in other instances of the code where the check was made correctly, such as in the [stakeMunchable](https://github.com/code-423n4/2024-07-munchables/blob/94cf468aaabf526b7a8319f7eba34014ccebe7b9/src/managers/LandManager.sol#L146) function

As a result, the plot with an ID equal to the total number of plots will be considered a valid plot, which it should not be.

## Tools Used
Manual Review

## Recommended Mitigation Steps
Update the mentioned check in `_farmPlots` to the following:
```diff
+ if (_getNumPlots(landlord) <= _toiler.plotId)
- if (_getNumPlots(landlord) < _toiler.plotId) {
    timestamp = plotMetadata[landlord].lastUpdated;
    toilerState[tokenId].dirty = true;
}
```

# H-02 Critical states are not updated when transferrnig tokens to new plot

## Impact

The [_farmPlots](https://github.com/code-423n4/2024-07-munchables/blob/94cf468aaabf526b7a8319f7eba34014ccebe7b9/src/managers/LandManager.sol#L232-L310) function is executed during every toiling operation on a plot (staking, unstaking, and transferring a token to a new plot). The function checks if the plotId associated with the ToilerState exceeds the number of available plots for the landlord. If so, it sets the `dirty` status to true:
```solidity
function _farmPlots(address _sender) internal {
    // ...

    for (uint8 i = 0; i < staked.length; i++) {
        timestamp = block.timestamp;
        tokenId = staked[i];
        _toiler = toilerState[tokenId];
        if (_toiler.dirty) continue; // <------------ skip the reward if already dirty
        landlord = _toiler.landlord;
        
        if (_getNumPlots(landlord) < _toiler.plotId) { 
             timestamp = plotMetadata[landlord].lastUpdated;
            toilerState[tokenId].dirty = true; // <------------ set dirty to true if the plot becomes invalid
        }

        // ...
    }
}
```

If the `dirty` status is set to true, it means that the current plot is no longer valid (so, schnibbles will not be farmed) and the token should be moved to a new valid plot to start farming again. And to transfer the token to a new valid plot, [transferToUnoccupiedPlot](https://github.com/code-423n4/2024-07-munchables/blob/94cf468aaabf526b7a8319f7eba34014ccebe7b9/src/managers/LandManager.sol#L199-L226) should be called. The issue is that the function does not update the `dirty` and `lastToildDate` states:
```solidity
function transferToUnoccupiedPlot(
    uint256 tokenId,
    uint256 plotId
) external override forceFarmPlots(msg.sender) notPaused {
    (address mainAccount, ) = _getMainAccountRequireRegistered(msg.sender);
    ToilerState memory _toiler = toilerState[tokenId];
    uint256 oldPlotId = _toiler.plotId;
    uint256 totalPlotsAvail = _getNumPlots(_toiler.landlord);
    if (_toiler.landlord == address(0)) revert NotStakedError();
    if (munchableOwner[tokenId] != mainAccount) revert InvalidOwnerError();
    if (plotOccupied[_toiler.landlord][plotId].occupied)
        revert OccupiedPlotError(_toiler.landlord, plotId);
    if (plotId >= totalPlotsAvail) revert PlotTooHighError();

    toilerState[tokenId].latestTaxRate = plotMetadata[_toiler.landlord]
            .currentTaxRate;
     // @audit `toilerState[tokenId].lastToilDate` was not updated
    plotOccupied[_toiler.landlord][oldPlotId] = Plot({
        occupied: false,
        tokenId: 0
    });
    plotOccupied[_toiler.landlord][plotId] = Plot({
        occupied: true,
        tokenId: tokenId
    });

    // @audit `toilerState[tokenId].dirty` was not updated

    emit FarmPlotLeave(_toiler.landlord, tokenId, oldPlotId);
    emit FarmPlotTaken(toilerState[tokenId], tokenId);
}
```

We can see that the function does not update the `dirty` status back to false if it was set to true. Therefore, even if the user transfers his token to a new valid plot because of the dirty status, the token will still be considered dirty and will not be able to farm schnibbles rewards.`lastToilDate` was not updated either, and it still points to the previous toiling action. 

## Tools Used
Manual Review

## Recommended Mitigation Steps
Consider updating the `lastToilDate` state and the `dirty` status if it was true. Below is a suggestion for an updated code of `transferToUnoccupiedPlot`:
```diff
function transferToUnoccupiedPlot(
    uint256 tokenId,
    uint256 plotId
) external override forceFarmPlots(msg.sender) notPaused {
    (address mainAccount, ) = _getMainAccountRequireRegistered(msg.sender);
    ToilerState memory _toiler = toilerState[tokenId];
    uint256 oldPlotId = _toiler.plotId;
    uint256 totalPlotsAvail = _getNumPlots(_toiler.landlord);
    if (_toiler.landlord == address(0)) revert NotStakedError();
    if (munchableOwner[tokenId] != mainAccount) revert InvalidOwnerError();
    if (plotOccupied[_toiler.landlord][plotId].occupied)
        revert OccupiedPlotError(_toiler.landlord, plotId);
    if (plotId >= totalPlotsAvail) revert PlotTooHighError();

    toilerState[tokenId].latestTaxRate = plotMetadata[_toiler.landlord]
            .currentTaxRate;
+    toilerState[tokenId].lastToildDate = block.timestamp;
    plotOccupied[_toiler.landlord][oldPlotId] = Plot({
        occupied: false,
        tokenId: 0
    });
    plotOccupied[_toiler.landlord][plotId] = Plot({
        occupied: true,
        tokenId: tokenId
    });

+    if (toilerState[tokenId].dirty) toilerState[tokenId].dirty = false;
    emit FarmPlotLeave(_toiler.landlord, tokenId, oldPlotId);
    emit FarmPlotTaken(toilerState[tokenId], tokenId);
}
```

# M-01 Munchables can be staked with 0% Tax rate

## Impact
Players call [stakeMunchable](https://github.com/code-423n4/2024-07-munchables/blob/94cf468aaabf526b7a8319f7eba34014ccebe7b9/src/managers/LandManager.sol#L131-L168) to stake their Munchables on a plot. The function updates the `toilerState` of the NFT, and more particularly sets the `latestTaxRate` of the NFT's toiler state which affects the schnibbles allocation. 

The issue is that the function does not check if the plot metadata was set for the landlord or not:
```solidity
function stakeMunchable(
    address landlord,
    uint256 tokenId,
    uint256 plotId
) external override forceFarmPlots(msg.sender) notPaused {
    (address mainAccount, ) = _getMainAccountRequireRegistered(msg.sender);
    if (landlord == mainAccount) revert CantStakeToSelfError();
    if (plotOccupied[landlord][plotId].occupied)
        revert OccupiedPlotError(landlord, plotId);
    if (munchablesStaked[mainAccount].length > 10)
         revert TooManyStakedMunchiesError();
    if (munchNFT.ownerOf(tokenId) != mainAccount)
        revert InvalidOwnerError();

    uint256 totalPlotsAvail = _getNumPlots(landlord);
    if (plotId >= totalPlotsAvail) revert PlotTooHighError();

    if (
        !munchNFT.isApprovedForAll(mainAccount, address(this)) &&
        munchNFT.getApproved(tokenId) != address(this)
    ) revert NotApprovedError();
    munchNFT.transferFrom(mainAccount, address(this), tokenId);

    plotOccupied[landlord][plotId] = Plot({
        occupied: true,
        tokenId: tokenId
    });

    munchablesStaked[mainAccount].push(tokenId);
    munchableOwner[tokenId] = mainAccount;

    toilerState[tokenId] = ToilerState({
        lastToilDate: block.timestamp,
        plotId: plotId,
        landlord: landlord,
        latestTaxRate: plotMetadata[landlord].currentTaxRate, // @audit No check if the metadata was set
        dirty: false
});
```
If the plot metadata for the landlord was not set, the tax rate of the toiler state will be set to the default zero-value. As consequence, when the next toiling happens (which might take too long by the users's intention), the `forceFarmPlots` modifier will calculate the schnibbles rewards of the users [with 0% tax](https://github.com/code-423n4/2024-07-munchables/blob/94cf468aaabf526b7a8319f7eba34014ccebe7b9/src/managers/LandManager.sol#L287-L289):
```solidity
function _farmPlots(address _sender) internal {
    (
        address mainAccount,
        MunchablesCommonLib.Player memory renterMetadata
    ) = _getMainAccountRequireRegistered(_sender);
    // ... 
        schnibblesTotal =
                (timestamp - _toiler.lastToilDate) *
                BASE_SCHNIBBLE_RATE;
        schnibblesTotal = uint256((int256(schnibblesTotal) +(int256(schnibblesTotal) * finalBonus)) / 100);
        schnibblesLandlord = (schnibblesTotal * 
                            _toiler.latestTaxRate)  // @audit `latestTaxRate` will be zero
                            / 1e18;

        toilerState[tokenId].lastToilDate = timestamp;
        toilerState[tokenId].latestTaxRate = plotMetadata[_toiler.landlord].currentTaxRate;

        renterMetadata.unfedSchnibbles += (schnibblesTotal -schnibblesLandlord);

        landlordMetadata.unfedSchnibbles += schnibblesLandlord;
        // ...
    }
    accountManager.updatePlayer(mainAccount, renterMetadata);
}
```
We can see that the calculation of the landlord schnibbles `schnibblesLandlord` will be evaluated to zero because the toiler's `latestTaxRate` was set to zero upon the stake, **which makes the user earns schnibbles rewards with 0% tax**.

**Notice that it is not intended for the tax rate to be zero because it is bounded between `MIN_TAX_RATE` and `MAX_TAX_RATE` when calling `updateTaxRate`**
## Proof of Concept
Consider the following scenario that demonstrates the previously mentioned issue:
- There is a landlord `L` with 5 available plots. The plot metadata was not set for the landlord `L`
- Bob stakes his Munchables token on `L` at any given valid plot (let it be `1`). The [toilerState[Bob's token] will be updated accordingly](https://github.com/code-423n4/2024-07-munchables/blob/94cf468aaabf526b7a8319f7eba34014ccebe7b9/src/managers/LandManager.sol#L162-L168) and notice that `latestTaxRate` will be [set to zero](https://github.com/code-423n4/2024-07-munchables/blob/94cf468aaabf526b7a8319f7eba34014ccebe7b9/src/managers/LandManager.sol#L166) because plot metadata was not set for `L`
- Time passes, and Bob toils his staked plot, for example he wants to unstake his Munchables, so he calls `unstakeMunchable`, the modifier `forceFarmPlots` will be executed to farm schnibbles points to Bob for the time passed
    - The exeuction will come to calculate the schnibbles that will be allocated for landlore `L`: `schnibblesLandlord = (schnibblesTotal * _toiler.latestTaxRate) / 1e18;`, which will be evaluated to zero because `_toiler.latestTaxRate` was set to zero
    - Bob's schnibbles rewards will be fully farmed with 0% tax: `renterMetadata.unfedSchnibbles += (schnibblesTotal - schnibblesLandlord);`
## Tools Used
Manual Review

## Recommended Mitigation Steps
Consider preventing users from staking if the plot metadata was not set for the landlord. Below is a suggestion for an updated code of `stakeMunchable`:
```diff
function stakeMunchable(
    address landlord,
    uint256 tokenId,
    uint256 plotId
) external override forceFarmPlots(msg.sender) notPaused {
    (address mainAccount, ) = _getMainAccountRequireRegistered(msg.sender);
    if (landlord == mainAccount) revert CantStakeToSelfError();
    if (plotOccupied[landlord][plotId].occupied)
        revert OccupiedPlotError(landlord, plotId);
    if (munchablesStaked[mainAccount].length > 10)
        revert TooManyStakedMunchiesError();
    if (munchNFT.ownerOf(tokenId) != mainAccount)
        revert InvalidOwnerError();

    uint256 totalPlotsAvail = _getNumPlots(landlord);
    if (plotId >= totalPlotsAvail) revert PlotTooHighError();

    if (
        !munchNFT.isApprovedForAll(mainAccount, address(this)) &&
        munchNFT.getApproved(tokenId) != address(this)
    ) revert NotApprovedError();
+    if (plotMetadata[landlord].currentTaxRate == 0) revert PlotMetaDataNotSet();
    munchNFT.transferFrom(mainAccount, address(this), tokenId);

    plotOccupied[landlord][plotId] = Plot({
        occupied: true,
        tokenId: tokenId
    });

    munchablesStaked[mainAccount].push(tokenId);
    munchableOwner[tokenId] = mainAccount;

    toilerState[tokenId] = ToilerState({
        lastToilDate: block.timestamp,
        plotId: plotId,
        landlord: landlord,
        latestTaxRate: plotMetadata[landlord].currentTaxRate,
        dirty: false
    });

    emit FarmPlotTaken(toilerState[tokenId], tokenId);
}
```