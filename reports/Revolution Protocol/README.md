# Vulnerability details

## Impact
The `buyToken` function on `ERC20TokenEmitter.sol` enables anyone to purchase the ERC20 governance token anytime. A portion of the value spent on buying the ERC20 tokens is paid to protocol rewards contract. The value remaining after protocol rewards is shared among the treasury and creators. However, the sum of percentages used to calculate the shares does not equal to 100% which leads to the fact that it is guaranteed that on each call to the `buyToken` function, a portion of `msg.value` will be locked in the contract. The amount that will be locked depends on 2 protocol parameters (`creatorRate` and `entropyRate`) and the ETH value sent by the buyer. The higher the `creatorRate` and the lower the `entropyRate`, the higher the amount of ETH locked in the contract. The amount locked can even attend +90% of the total value sent by the buyer as demonstrated in the PoC section. The current implementation has no mechanism for refunding the locked value.

## Proof of Concept
The following test confirms the fact that a high amount of ETH sent by the buyer can be locked. The test demonstrates that +90% of ETH sent by the buyer will be locked inside the smart contract if the following protocol parameters are set:

- `creatorRate` = 9530
- `entropyRate` = 96
- with 0.0.0000028 ETH as `msg.value` sent by the buyer

These parameter values were found through the Fuzzing test.
```solidity
function testMostOfMsgValueCanBeLockedWhenBuyingToken() public {
        uint valueToSend = 2814749767106; // ~ 0.0.0000028 ETH
        uint creatorRate = 9530;
        uint entropyRate = 96;

        vm.startPrank(address(dao));
        // Set creator and entropy rates
        erc20TokenEmitter.setCreatorRateBps(creatorRate);
        erc20TokenEmitter.setEntropyRateBps(entropyRate);
        assertEq(erc20TokenEmitter.creatorRateBps(), creatorRate, "Creator rate not set correctly");
        assertEq(erc20TokenEmitter.entropyRateBps(), entropyRate, "Entropy rate not set correctly");

        // Setup for buying token
        address[] memory recipients = new address[](1);
        recipients[0] = address(1); // recipient address

        uint256[] memory bps = new uint256[](1);
        bps[0] = 10000; // 100% of the tokens to the recipient

        erc20TokenEmitter.setCreatorsAddress(address(80));

        // Perform token purchase
        vm.startPrank(address(this));
        erc20TokenEmitter.buyToken{ value: valueToSend }(
            recipients,
            bps,
            IERC20TokenEmitter.ProtocolRewardAddresses({
                builder: address(0),
                purchaseReferral: address(0),
                deployer: address(0)
            })
        );
        
        // audit-danger +90% of value sent by the buyer will be locked!!
        assertGt(address(erc20TokenEmitter).balance , (valueToSend * 90)/100);
}
```
```bash
Running 1 test for test/token-emitter/ERC20TokenEmitter.t.sol:ERC20TokenEmitterTest
[PASS] testMostOfMsgValueCanBeLockedWhenBuyingToken() (gas: 489894)
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 7.85ms
 
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Tools Used
Manual Analysis, Fuzz testing

## Recommended Mitigation Steps
Consider refunding the excess ETH at the end of the transaction. Below is a suggestion of an updated code of the `buyToken` function:
```solidity
function buyToken(
        ...

        //Transfer ETH to treasury and update emitted
        emittedTokenWad += totalTokensForBuyers;
        if (totalTokensForCreators > 0) emittedTokenWad += totalTokensForCreators;

        //Deposit funds to treasury
        (bool success, ) = treasury.call{ value: toPayTreasury }(new bytes(0));
        require(success, "Transfer failed.");

        //Transfer ETH to creators
        if (creatorDirectPayment > 0) {
            (success, ) = creatorsAddress.call{ value: creatorDirectPayment }(new bytes(0));
            require(success, "Transfer failed.");
        }

        // audit-fix get the excessEth and refund it
        uint excessEth = msgValueRemaining - toPayTreasury - creatorDirectPayment;
        refund(remainingETH);
```