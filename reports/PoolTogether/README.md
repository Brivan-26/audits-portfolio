# H-01 Wrong update of `yieldFeeBalance` on `PrizeVault::claimYieldFeeShares`

On the `PrizeVault` contract, the yield fee recipient calls the `claimYieldFeeShares` function to receive yield fee shares
```js
function claimYieldFeeShares(uint256 _shares) external onlyYieldFeeRecipient {
    if (_shares == 0) revert MintZeroShares();

    uint256 _yieldFeeBalance = yieldFeeBalance;
    if (_shares > _yieldFeeBalance) revert SharesExceedsYieldFeeBalance(_shares, _yieldFeeBalance);

    yieldFeeBalance -= _yieldFeeBalance; // @audit wrong update, 'yieldFeeBalance` = 0 now

    _mint(msg.sender, _shares);

    emit ClaimYieldFeeShares(msg.sender, _shares);
}
```
You can see that the update of `yieldFeeBalance` is updated wrongly in which it is set to zero while it should be subtracted from `_shares`

## Impact
If the fee recipient wants to receive a portion of the fee shares, the left shares will not be redeemable because the state is set to zero.

## Proof of Concept
Consider the following example:
- Available yield fee balance is `30`
- Fee recipient wants to receive `20` shares, he calls `claimYieldFeeShares(20)`
- `yieldFeeBalance` is updated as follows: `yieldFeeBalance -= _yieldFeeBalance = 0`
- The fee recipient can not claim the `10` shares left.
## Tools Used
Manual Review
## Recommended Mitigation Steps
Two possible approaches to solve the issue:
1. Change the `yieldFeeBalance` update to be subtracted from `_shares`
```diff
function claimYieldFeeShares(uint256 _shares) external onlyYieldFeeRecipient {
    if (_shares == 0) revert MintZeroShares();

    uint256 _yieldFeeBalance = yieldFeeBalance;
    if (_shares > _yieldFeeBalance) revert SharesExceedsYieldFeeBalance(_shares, _yieldFeeBalance);

-    yieldFeeBalance -= _yieldFeeBalance; 
+    yieldFeeBalance -= _shares; 
    _mint(msg.sender, _shares);

    emit ClaimYieldFeeShares(msg.sender, _shares);
}
```
2. Update the logic so that the fee recipient redeems the whole yield fee balance (remove the `_shares` from function's params)
```diff
function claimYieldFeeShares(uint256 _shares) external onlyYieldFeeRecipient {
    if (_shares == 0) revert MintZeroShares();

    uint256 _yieldFeeBalance = yieldFeeBalance;
    if (_shares > _yieldFeeBalance) revert SharesExceedsYieldFeeBalance(_shares, _yieldFeeBalance);

    yieldFeeBalance -= _yieldFeeBalance

-    _mint(msg.sender, _shares);
+    _mint(msg.sender, _yieldFeeBalance);
    emit ClaimYieldFeeShares(msg.sender, _shares);
}
```

# M-01 `PrizeVault::withdraw` does not protect users from burning too many shares in case of lossy situation
The function `PrizeVault::withdraw` withdraws `assets` by burning `shares`
```js
function withdraw(
        uint256 _assets,
        address _receiver,
        address _owner
    ) external returns (uint256) {
    uint256 _shares = previewWithdraw(_assets);
    _burnAndWithdraw(msg.sender, _receiver, _owner, _shares, _assets);
    return _shares;
}
```
Assets and shares are meant to be 1:1, but if the vault controls less assets than what has been deposited, a share will be worth aproportional amount of the total assets:
```js
if (_totalAssets >= totalDebt_) {
    return _assets;
} else {
    // Follows the inverse conversion of `convertToAssets`
    return _assets.mulDiv(totalDebt_, _totalAssets, Math.Rounding.Up);
}
```
The issue is that the `withdraw` function does not protect users from burning too many shares when the contract controls less assets than the debts.

## Impact
Too many shares can be burned that are not expected by users.

## Tools Used
Manual Review
## Recommended Mitigation Steps
Consider checking for max shares to be burned when calling `withdraw`. Below is the updated code:
```diff
function withdraw(
    uint256 _assets,
    address _receiver,
    address _owner,
+   uint256 _maxSharesOut
) external returns (uint256) {
    uint256 _shares = previewWithdraw(_assets);
+    if (_shares > _maxSharesOut) revert burningSharesExceedsMax();
    _burnAndWithdraw(msg.sender, _receiver, _owner, _shares, _assets);
    return _shares;
}
```